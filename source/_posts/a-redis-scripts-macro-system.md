title: A Redis scripts macro system
date: 2014-03-28
tags: Redis, Node.js
category: Work.en/Backend
---

When designing *Caaards.com*, I had decided to use Redis as main database and to
 manually flush cold data to the disk. Later, I noticed Redis's support for Lua
 scripts.

It works simply: just pass a piece of Lua script with some arguments. To access
those arguments (keys and values) you could access two **global** arrays,
``KEYS`` and ``ARGV``. However, it has some special limits:

1. keys are distinguished from other arguments so that your script could be sent
 to the right node in a cluster. When you pass arguments, you need to specify
the number of keys.

2. Each script should be a pure function so Redis could keep data consistence
between master and slave. What does this imply? You could not create global
variables, call random generators or get the time.

3. You could not call other scripts from your script. That didn't make sense to
me. I asked a [question on Stack Overflow][1]. However, my question is redundant
and *Josiah* answered the correct question [here][2]. It is said that ``KEYS``
and ``ARGV`` are two **globals**. If we need to call another script, Redis will
have to maintain another instance, which might be too heavy (the author boasted
 that script module only used three hundred lines).

The third one is serious to me, as my design is to write as much code base
in Lua as possible. If I couldn't reuse code, it will be a catastrophe.

Let's see an example any way:

```Lua
local intervals
if redis.call('EXISTS', KEYS[4]) == 1 then
  intervals = redis.call('GET', KEYS[4])
else
  intervals = {5, 15, 60, 180, 720, 2880, 5760, 11520, 23040}
end

local star = tonumber(redis.call('GET', KEYS[3]))
local time = ARGV[1] + intervals[star] * (1 + ARGV[2]) * 60 * 1000

return redis.call('ZADD', KEYS[1], time, KEYS[2])
```

In Lua, array index counts from 1 instead of 0.

Another issue is that code readability decreases due to meaningless variable
name.

---

I decided to introduce a simple macro system. Why simple? Because I had never
implemented a parser. Therefore, all macros are wrapped with '%'. Scripts are
stored in files and each file has a metadata table. The table records variables
involved, the script name to other scripts, and the command name to clients. The
 above code could be rewritten like this:

```Lua
--[[
name: quiz_update
keys: %quizset%, %word_key%, %star_key%, %interval_list%
argv: %now_time%, %coefficient%
]]--

local intervals
if redis.call('EXISTS', %interval_list%) == 1 then
  intervals = redis.call('GET', %interval_list%)
else
  intervals = {5, 15, 60, 180, 720, 2880, 5760, 11520, 23040}
end

local star = tonumber(redis.call('GET', %star_key%))
local time = %now_time% + intervals[star] * (1 + %coefficient%) * 60 * 1000

return redis.call('ZADD', %quizset%, time, %word_key%)
```

Does it look much nicer?

How to call this script from other ones?

```Lua
%quiz_update%({ %quizset%, %word_key%, %star_key%, %interval_list% },
              { %now_time%, %coefficient% })
```

I have finished this project but no tests have been done. I plan to publish it
in one week.

[1]:http://stackoverflow.com/questions/22432495/why-does-redis-forbid-user-script-to-call-other-scripts-how-to-keep-lua-scripts

[2]:http://stackoverflow.com/questions/21718277/is-it-possible-to-call-lua-functions-defined-in-other-lua-scripts-in-redis/22599862
