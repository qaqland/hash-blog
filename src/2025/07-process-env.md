# 进程与环境变量

2025年5月18日 在想办法用环境变量做程序的配置系统

## 数据结构

理论上讲应该采用 HashMap 的数据格式保存，以实现 O(1) 的访问速度。

```c
#include <unistd.h>

char **__environ = 0;
weak_alias(__environ, ___environ);
weak_alias(__environ, _environ);
weak_alias(__environ, environ);

// https://git.musl-libc.org/cgit/musl/tree/src/env/__environ.c
```

但实际使用形如 `KEY=VALUE` 的字符串数组。

## 系统调用

对环境变量的读和写均发生在用户态：

```c
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

char *getenv(const char *name)
{
    size_t l = __strchrnul(name, '=') - name;
    if (l && !name[l] && __environ)
        for (char **e = __environ; *e; e++)
            if (!strncmp(name, *e, l) && l[*e] == '=')
                return *e + l+1;
    return 0;
}

// https://git.musl-libc.org/cgit/musl/tree/src/env/getenv.c
```

仅初始化与系统调用有关：

```c
#include <unistd.h>
#include "syscall.h"

int execve(const char *path, char *const argv[], char *const envp[])
{
    /* do we need to use environ if envp is null? */
    return syscall(SYS_execve, path, argv, envp);
}

// https://git.musl-libc.org/cgit/musl/tree/src/process/execve.c
```

## Bash

在 Bash 中很坏，“环境变量”只是普通变量的一个属性，一般用 `export` 关键字标识

```c
#define att_exported    0x0000001   /* export to environment */
...
#define att_local       0x0000020   /* variable is local to a function */

// bash/variables.h

/* An array which is passed to commands as their environment.  It is
   manufactured from the union of the initial environment and the
   shell variables that are marked for export. */
char **export_env = (char **)NULL;

// bash/variables.c
```

所以会出现以下情况，环境变量随变量修改而改变：

```bash
export foo=bar
foo=barbar

sh -c 'echo $foo'
# output: barbar
```

其它对环境变量的构建和修改与普通函数无异

```c
static inline char *
mk_env_string (name, value, attributes)
     const char *name, *value;
     int attributes;
{
    ...
    {
      p = (char *)xmalloc (2 + name_len + value_len);
      memcpy (p, name, name_len);
      q = p + name_len;
    }
    q[0] = '=';

	memcpy (q + 1, value, value_len + 1);
}

// bash/variables.c


int
shell_execve (command, args, env)
     char *command;
     char **args, **env;
{
    ...
    execve (command, args, env);
}

// bash/execute_cmd.c
```
