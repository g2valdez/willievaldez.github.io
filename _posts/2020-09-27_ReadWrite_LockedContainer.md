# 2020-09-27 A Simple Read-Write Locked Container
## Thread Contention in C++
A huge sticking point in multi-threaded programming is concurrent access of the same variable.
The simple solution to this problem is to create a mutex, and lock the mutex in any part of code
that touches your "danger" variable. A slightly more sophisticated access pattern is the 
["read-write" lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock). Essentially, 
any number of threads can read from a variable at the same time, but only one thread can write 
to a variable at any time. When the "write-lock" is acquired, "read-locks" cannot be acquired either.

## The Problem with the Solution
The perfect programmer will always accompany any accesses of this "danger" variable with the proper
lock acquisition. The imperfect programmer (read: me) might miss one or two areas of code that access
the variable. So, I decided to dummy-proof the read-write lock access pattern.

## The Implementation
I wanted to write a container that can hold any data type, and exposes a "Get" (read) and an "Access" (write).
The object returned by "Get" will acquire a read lock throughout the lifetime of its existence, and likewise,
the object returned by "Access" will acquire a write lock.

Here's LockedContainer.h
```
#pragma once
#include <shared_mutex>
#include <mutex>

template <typename T>
class LockedContainer
{
public:
    LockedContainer() = default;
    LockedContainer(const T& data) : m_data(data) {};

    class ReadContainer
    {
    public:
        ReadContainer(const T& data, std::shared_mutex& mutex) : m_lock(mutex), m_data(data) {};
        const T& operator->() { return m_data; };
        const T& operator*() { return m_data; };
    private:
        const T& m_data;
        std::shared_lock<std::shared_mutex> m_lock;
    };

    class WriteContainer
    {
    public:
        WriteContainer(T& data, std::shared_mutex& mutex) : m_lock(mutex), m_data(data) {};
        T& operator->() { return m_data; };
        T& operator*() { return m_data; };
    private:
        T& m_data;
        std::unique_lock<std::shared_mutex> m_lock;
    };

    ReadContainer Get()
    {
        return ReadContainer(m_data, m_mutex);
    }

    WriteContainer Access()
    {
        return WriteContainer(m_data, m_mutex);
    }

    void operator=(const T& val)
    {
        *(Access()) = val;
    }

private:
    T m_data;
    std::shared_mutex m_mutex;
};
```

Awesome! Now, if you wanted to declare a lock-protected int, as a member variable of a class, you declare it like this:
```
LockedContainer<int> m_sharedInt;
```
And if you want to read-lock a section of your code, you call this:
```
LockedContainer<int>::ReadContainer container = m_sharedInt.Get();
```
Similarly, a write-lock would look like this:
```
LockedContainer<int>::WriteContainer container = m_sharedInt.Access();
```
Or, if you were simply setting the variable, you can do a quick write lock like this:
```
m_sharedInt = 0;
```

## Testing
I wrote a quick main function that tested this access pattern. It spawns 200 threads, every 10 of which are write functions. The rest are read functions.
At the end, I print the contents of the variable, and the time (in microseconds) it took to acquire the lock.
```
#include "LockedContainer.h"
#include <iostream>
#include <thread>
#include <chrono>
#include <string>

#define NUM_ITERATIONS 200
std::string results[NUM_ITERATIONS];

LockedContainer<int> sharedInt;

void ReadFunc(int i)
{
    auto start = std::chrono::high_resolution_clock::now();
    LockedContainer<int>::ReadContainer container = sharedInt.Get();
    auto end = std::chrono::high_resolution_clock::now();
    results[i] = "Read=" + std::to_string(*container) + " (" + std::to_string(std::chrono::duration_cast<std::chrono::microseconds>(end - start).count()) + " us)";
}

void WriteFunc(int i)
{
    auto start = std::chrono::high_resolution_clock::now();
    LockedContainer<int>::WriteContainer container = sharedInt.Access();
    auto end = std::chrono::high_resolution_clock::now();
    (*container)++;
    results[i] = "Write=" + std::to_string(*container) + " (" + std::to_string(std::chrono::duration_cast<std::chrono::microseconds>(end - start).count()) + " us)";
}

int main()
{
    sharedInt = 0;
    std::thread threads[NUM_ITERATIONS];

    for (int i = 0; i < NUM_ITERATIONS; i++)
    {
        if (i % 10 == 0)
        {
            threads[i] = std::thread(WriteFunc, i);
        }
        else
        {
            threads[i] = std::thread(ReadFunc, i);
        }
    }

    for (int i = 0; i < NUM_ITERATIONS; i++)
    {
        threads[i].join();
        std::cout << results[i].c_str() << std::endl;
    }

    return 0;
}
```

All in all, looks pretty good, and is easy to use. I'm happy with how this little escapade turned out.
