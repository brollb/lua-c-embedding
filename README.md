# luajit-c-embedding

Here are some notes/instructions for embedding the LuaJIT in C. Initially, we follow along with [Marek Vavrusa's blog post](https://en.blog.nic.cz/2015/08/12/embedding-luajit-in-30-minutes-or-so/). Later, we move beyond and see how to interact with things in Torch.

### Prerequisites

If [LuaJIT](http://luajit.org/download.html) and [Luarocks](https://github.com/keplerproject/luarocks/wiki/Download) aren't already installed, you'll need to install them. I used LuaJIT 2.1.0-beta2. What follows assumes that you've installed them from source or via a package manager like Homebrew. For compilation, I've used gcc, but have also checked that clang should work without issue. On macOS clang might be a better option.

## Hello, World!

Let's start by creating a Lua script named hello.lua which contains ```print('Hello, World!')```. Next, create a C file hello-luajit.c which looks like this (modified from the above blog post):

```
#include <stdio.h>
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include "luajit.h"

int main(int argc, char *argv[])
{
  int status;
  lua_State *L;
  
  L = luaL_newstate(); // open Lua
  if (!L) {
    return -1; // Checks that Lua started up
  }
  
  luaL_openlibs(L); // load Lua libraries
  if (argc > 1) {
    status = luaL_loadfile(L, argv[1]);  // load Lua script
    int ret = lua_pcall(L, 0, 0, 0); // tell Lua to run the script
    if (ret != 0) {
      fprintf(stderr, "%s\n", lua_tostring(L, -1)); // tell us what mistake we made
      return 1;
    }
  }

  lua_close(L); // Close Lua
  return 0;
}
```

To compile the C, use:

```gcc hello-luajit.c -pagezero_size 10000 -image_base 100000000 -I/usr/local/include/luajit-2.1 -L/usr/local/lib -lluajit -o hello.out```, noting that ```-pagezero_size 10000 -image_base 100000000``` only needs to be included on macOS systems. Also note how the LuaJIT include directory has been included in the compiler search path. 

You can now execute the Lua script with ```./hello.out hello.lua```.

### Understanding the C Code

To interact with Lua from C we will use [Lua's C API](http://www.lua.org/pil/24.1.html). See http://pgl.yoyo.org/luai/i/_ for great documentation on the functions in Lua's C API. Let's look at the each part of hello-luajit.c. 

- ```luaL_newstate```: Starts up Lua and returns a new Lua state
- ```luaL_openlibs```: loads the Lua standard libraries
- ```luaL_loadfile```: Loads the Lua script
- ```lua_pcall```: runs the Lua script and returns an error code
- ```lua_close```: closes Lua

## Using the Lua Stack

Lua uses a stack to exchange values with C. To get a value from Lua, you call Lua to push the value onto the stack. To pass a value from C to Lua, you call Lua to push the value onto the stack and then pop the value. When pushing a value onto the stack, it's important to know if the stack has room for the new value. The lua-users wiki has [a nice example of this](http://lua-users.org/wiki/SimpleLuaApiExample). Let's look a couple of examples.

### Example 1

Suppose we want to compute the factorial of a number in Lua and make the result of the computation available in C. The Lua code is in the file factorial.lua:

```
function factorial(n)
  local result = 1
  for i = 1, n do
    result = result * i
  end
  return result
end
```

From C, we'll call factorial.lua and then get the result from the Lua stack and print to stdout.

```
#include <stdio.h>
#include <stdlib.h>
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include "luajit.h"

int main(int argc, char *argv[])
{
  lua_State *L;
  
  L = luaL_newstate(); // open Lua 
  luaL_openlibs(L); // load Lua libraries
  int n = atoi(argv[1]);

  luaL_loadfile(L, "factorial.lua");
  lua_pcall(L, 0, 0, 0); // Execute script once to create and assign functions
	
  lua_getglobal(L, "factorial"); // function to be called
  lua_pushnumber(L, n); // push argument
	
  if (lua_pcall(L, 1, 1, 0) != 0) // 1 argument, 1 return value
  {
    fprintf(stderr, "%s\n", lua_tostring(L, -1));
    return 1;
  }
  
  int result = lua_tonumber(L, -1);
  lua_pop(L, 1); // pop returned value
  printf("%d! is %d\n", n, result);  
    
  lua_close(L); 
  return 0;
}
```

This example is pretty simple:

1. First, we load the Lua file containing the factorial function
- Call ```lua_pcall``` to execute the script and create the functions
- Call ```lua_getglobal``` to specify which function you wish to call
- Push the function arguments to the stack, in order
- Call ```lua_pcall``` again, specifying the number of arguments and return values
- Use ```lua_tointeger``` to get the return value from the top of the stack

You'll notice that at the end we pop the returned value from the stack with ```lua_pop```. For such a simple example, this is not needed since we're killing Lua afterwards, but for more complex uses you'll definitely want to clean up the stack before moving on.

You can compile with ```gcc factorial-luajit.c -pagezero_size 10000 -image_base 100000000 -I/usr/local/include/luajit-2.1 -L/usr/local/lib -lluajit -o fact.out```. You can run with ```./fact.out <some integer>```

### Example 2

Now we'll use Torch to compute the trace of a matrix. For this, we'll simply execute a Lua script that requires Torch:

```
require('torch')

function cosine(theta)
  return torch.cos(theta)
end
```

This example is functionally identical to the previous one. You should be able to do this on your own now. If you run into any trouble, just look at torch-test.c.

## Using Lua From Java

See jni_notes.md for examples of how to use LuaJIT from Java.



