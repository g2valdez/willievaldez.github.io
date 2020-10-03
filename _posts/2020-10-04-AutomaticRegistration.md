---
layout: post
title: (WIP) Class Registration During Static Init
subtitle: Tricking C++ to do your bidding
tags: [cpp, oop]
---

Have you ever wanted to identify a class simply by supplying a string?
You and me both, buddy. I was working on a personal game project and I wanted
to define a level by feeding the program a CSV file containing the spatial layout.
I identify different types of enemy spawners in the CSV file with human readable names.
Each name correlates to an explicit AI class derived from a base class `Enemy`.


## The Wrong Way(s) To Do It
When loading the CSV, cache the spawn type in the spawner as a member variable `std::string m_spawnType`.
 Inside of `Spawner::Spawn()`, you can implement this god awful monstrosity:
```cpp
std::shared_ptr<Enemy> Spawner::Spawn()
{
    if (m_spawnType == "Goomba")
    {
        return std::make_shared<Goomba>();
    }

    // ... lots of else ifs here
    
    else if (m_spawnType == "Koopa")
    {
        return std::make_shared<Koopa>();
    }
}
```
*note: the string definition could just as easily be converted to an enum to avoid the computationally expensive string compare*

How about we try to make it a little smarter? `Spawner` can keep a static map of strings to `Enemy` contructors.

```cpp
 static std::unordered_map<std::string, std::shared_ptr<Enemy>(*)()> s_spawnTypeMap;
```

This immensely simplifies (and optimizes) our `Spawner::Spawn()` function:
```cpp
std::shared_ptr<Enemy> Spawner::Spawn()
{
    std::shared_ptr<Enemy> spawnedEnemy;

    auto& foundEnemy = s_spawnTypeMap.find(m_spawnType);
    if (foundEnemy != s_spawnTypeMap.end())
    {
        spawnedEnemy = (*foundEnemy)();
    }

    return spawnedEnemy;
}
```

*BUT* we have to populate that map somehow, right? This usually leads to a big ugly function definition in a cpp file that looks like this:
```cpp
#include <Goomba.h>
// ... lots of includes here
#include <Koopa.h>

/*static*/ void Spawner::RegisterSpawnTypes()
{
    s_spawnTypeMap["Goomba"] = &std::make_shared<Goomba>;
    // ... lots of map insertions here
    s_spawnTypeMap["Koopa"] = &std::make_shared<Koopa>;
}
```

Then, before we load our level file, we can call `Spawner::RegisterSpawnTypes()`. Simple, right? WRONG. This implementation sucks because:
* Whenever you make a new Enemy class, you can't forget to add an entry in to RegisterSpawnTypes
* Modifying any derived Enemy header file forces you to recompile this source file EVERY TIME

## The Better Way To Do It

