---
layout: post
title: (WIP) Class Registration During Static Init
subtitle: Tricking C++ to do your bidding
tags: [cpp, oop]
---

Have you ever wanted to identify a class simply by supplying a string?
You and me both, buddy. I was working on a personal game project and I wanted
to define a level by feeding the program a simple CSV file containing a layout.
Say I had 10 different types of enemy spawners that I could place. I identify 
them in the csv file with human readable names. Each name correlates to an explicit
AI derived from a base class `Enemy`.


## The Wrong Way(s) To Do It
When loading the CSV, cache the spawn type in the spawner as a member variable `std::string m_spawnType`.
 Inside of `Spawner::Spawn()`, you can implement this god awful monstrosity
```cpp
Enemy* Spawner::Spawn()
{
	if (m_spawnType == "Goomba")
	{
		return new Goomba();
	}
	// ... lots of else ifs here
	else if (m_spawnType == "Koopa")
	{
		return new Koopa();
	}
}
```

How about we try to make it a little smarter? `Spawner` can keep a static map of ints to Enemy contructors.

```cpp
 static std::unordered_map<int, Enemy*(*)()>
```