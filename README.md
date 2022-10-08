# 实现一个Shell

## 简单熟悉下C

```c
int a = 10;
char b = 'a';
char* string = "我喜欢吃饭";
double money = 10.36;
float earnMoney(int day) {
  return day * 5.9;
}
char* sayword() {
  char* word = malloc(sizeof(char)*5);
  word[0] = 'a';
  word[1] = 'k';
  word[2] = 'e';
  word[3] = 'r';
  word[4] = '\0'; // 0 / NULL
  return word
}

////////////
// - unistd.h
// 等待线程信号返回
// WNOHANG	如果pid指定的子进程没有结束，则waitpid()函数立即返回0，而不是阻塞在这个函数上等待；如果结束了，则返回该子进程的进程号。
// WUNTRACED	如果子进程进入暂停状态，则马上返回。
pid_t (*waitpid)(pid_t, int* status, int option); // status获取同步内部状态；option [WNOHAND | WUNTRACED]
// 孵化、衍生一个新的process
pid_t (*fork)(void);
// 执行
int (*execvp)(const char* command, char* argv[]); // -1代表返回的错误
// 更新程序执行目录上下文
int (*chdir)(const char* path); // 成功则返回0；失败返回-1

////////////
// - string.h
// char*比较
int (*strcmp)(const char* s1, const char* s2); // 返回0代表s1 == s2
// 分解 .e.g. "node -v".split(" ") -> "node", "-v"
char* (*strtok)(char* __str, const char* __sep); // 返回当前分解出的单词

///////////
// - stdlib.h
// 分配内存
void* (*malloc)(size_t);
// 重新分配内存
void* (*realloc)(void* __ptr, size_t __size);

///////////
// - stdio.h
// 获取输入
char (*getchar)(void);
```

## 实战

malloc的堆数据一定记得清理。

```c
void aker_process() {
  char* line = aker_getLine();
  char** args = aker_getArgs(line);
  int status = aker_exec(args);

  free(args);
  free(line);
}
```

### 获取用户输入

设置buffer大小，当需要更多buffer时，再次申请扩容即可。

```c
#define AKER_LINE_BUFFER_SIZE 64
char* aker_getLine() {
  int bufferSize = AKER_LINE_BUFFER_SIZE, position = 0;
  char *line = malloc(sizeof(char) * bufferSize);
  printf("aker> ");
  while (1) {
    char c = getchar();
    if (c == EOF) {
      printf("退出中...\n");
      printf("欢迎下次使用\n");
      exit(EXIT_SUCCESS);
    } else if (c == '\n') {
      line[position] = '\0'; // 字符串结尾符
      return line;
    }
    line[position] = c;
    position++;

    if (position >= bufferSize) {
      bufferSize += AKER_LINE_BUFFER_SIZE;
      line = realloc(line, sizeof(char) * bufferSize);
      if (line == NULL) {
        printf("aker_getLine realloc error!\n");
        exit(EXIT_FAILURE);
      }
    }
  }
}
```

### 用户输入分解为多个字符

原理同上

```c
#define AKER_ARG_BUFFER_SIZE 20
#define AKER_ARG_SEP_TOKEN " \t\n\r"
char** aker_getArgs(char* line) {
  int position = 0, bufferSize = AKER_ARG_BUFFER_SIZE;
  char **args = malloc(sizeof(char*) * AKER_ARG_BUFFER_SIZE);
  char *token = strtok(line, AKER_ARG_SEP_TOKEN);

  while (token != NULL) {
    args[position] = token;
    position++;
    if (position >= bufferSize) {
      bufferSize += AKER_ARG_BUFFER_SIZE;
      args = realloc(args, sizeof(char*) * AKER_ARG_BUFFER_SIZE);
      if (!args) {
        printf("aker_getArgs realloc error!\n");
        exit(EXIT_FAILURE);
      }
    }
    token = strtok(NULL, AKER_ARG_SEP_TOKEN);
  }

  args[position] = NULL;
  return args;
}
```

### 创建进程执行分解的命令

```c
int aker_launch(char **args) {
  pid_t pid, wpid;
  int status;
  pid = fork();
  if (pid < 0) {
    puts("aker_launch fork error!");
    exit(EXIT_FAILURE);
  } else if (pid == 0) {
    if (execvp(args[0], args) == -1) {
      printf("aker child exec fail!\n");
      exit(EXIT_FAILURE);
      return 1;
    }
    exit(EXIT_SUCCESS);
  } else {
    do {
      wpid = waitpid(pid, &status, WUNTRACED);
    } while (WIFEXITED(status) == 0 && WIFSIGNALED(status) == 0);
    // WIFEXITED(status) 如果子进程正常结束，他就返回真；否则返回假；
    // WEXITSTATUS(status) 如果WIFEXITED(status)为真，则可以用该宏取得子进程exit()返回的结束代码。
    // WIFSIGNALED(status) 如果子进程因为一个未捕获的信号而终止，他就返回真；否则返回假；
    // WTERMSIG(status) 如果WIFSIGNALED(status)为真，则可以用该宏获得导致子进程终止的信号代码；
  }
  return 1;
}
```

### 执行`cd / exit` 命令

...完成内置命令

### 选取历史命令`^[[A / ^[[B`

...

## 完整实现:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <editline/readline.h>

int aker_cd(char** args) {
  if (args[1] == NULL) {
    printf("请输入路径!");
    return 1;
  }
  // int (*chdir)(const char* path);
  // 成功则返回0；失败返回-1
  if (chdir(args[1]) != 0) {
    perror("aker chdir()");
  }
  return 1;
}
int aker_exit(char** args) {
  printf("waiting for your next coming!");
  exit(EXIT_SUCCESS);
}
int aker_help(char** args) {
  printf("commands for a new shell!\n");
  printf("cd / exit / help / ... and more for discover!\n");
  return 1;
}

int (*aker_buildins[])(char** args) = {
  &aker_cd,
  &aker_help,
  &aker_exit
};

char* aker_buildinStr[] = {
  "cd",
  "help",
  "exit"
};
int aker_builtinSize() {
  return sizeof(aker_buildinStr) / sizeof(aker_buildinStr[0]);
}

int aker_launch(char **args) {
  pid_t pid, wpid;
  int status;
  pid = fork();
  if (pid < 0) {
    puts("aker_launch fork error!");
    exit(EXIT_FAILURE);
  } else if (pid == 0) {
    // int (*execvp)(const char* command, char* argv[]);
    // -1 代表返回的错误
    if (execvp(args[0], args) == -1) {
      printf("aker child exec fail!\n");
      exit(EXIT_FAILURE);
      return 1;
    }
    exit(EXIT_SUCCESS);
  } else {
    do {
      wpid = waitpid(pid, &status, WUNTRACED);
    } while (WIFEXITED(status) == 0 && WIFSIGNALED(status) == 0);
    // WIFEXITED(status) 如果子进程正常结束，他就返回真；否则返回假；
    // WEXITSTATUS(status) 如果WIFEXITED(status)为真，则可以用该宏取得子进程exit()返回的结束代码。
    // WIFSIGNALED(status) 如果子进程因为一个未捕获的信号而终止，他就返回真；否则返回假；
    // WTERMSIG(status) 如果WIFSIGNALED(status)为真，则可以用该宏获得导致子进程终止的信号代码；
  }
  return 1;
}

int aker_exec(char **args) {
  if (args[0] == NULL) {
    printf("you have no input anything!");
    return 1;
  }
  for (int i = 0; i < aker_builtinSize(); i++) {
    char *command = aker_buildinStr[i];
    // int (*strcmp)(const char* s1, const char* s2);
    // 相等0; 大于 0；小于 0；
    if (strcmp(command, args[0]) == 0) {
      return (aker_buildins[i])(args);
    }
  }
  return aker_launch(args);
}

#define AKER_ARG_BUFFER_SIZE 20
#define AKER_ARG_SEP_TOKEN " \t\n\r"
char** aker_getArgs(char* line) {
  int position = 0, bufferSize = AKER_ARG_BUFFER_SIZE;
  char **args = malloc(sizeof(char*) * AKER_ARG_BUFFER_SIZE);
  char *token = strtok(line, AKER_ARG_SEP_TOKEN);

  while (token != NULL) {
    args[position] = token;
    position++;
    if (position >= bufferSize) {
      bufferSize += AKER_ARG_BUFFER_SIZE;
      args = realloc(args, sizeof(char*) * AKER_ARG_BUFFER_SIZE);
      if (!args) {
        printf("aker_getArgs realloc error!\n");
        exit(EXIT_FAILURE);
      }
    }
    token = strtok(NULL, AKER_ARG_SEP_TOKEN);
  }

  args[position] = NULL;
  return args;
}

// 使用getchar 方法
#define AKER_LINE_BUFFER_SIZE 64
char* aker_getLine() {
  int bufferSize = AKER_LINE_BUFFER_SIZE, position = 0;
  char *line = malloc(sizeof(char) * bufferSize);
  printf("aker> ");
  while (1) {
    char c = getchar();
    if (c == EOF) {
      printf("退出中...\n");
      printf("欢迎下次使用\n");
      exit(EXIT_SUCCESS);
    } else if (c == '\n') {
      line[position] = '\0';
      return line;
    }
    line[position] = c;
    position++;

    if (position >= bufferSize) {
      bufferSize += AKER_LINE_BUFFER_SIZE;
      line = realloc(line, sizeof(char) * bufferSize);
      if (line == NULL) {
        printf("aker_getLine realloc error!\n");
        exit(EXIT_FAILURE);
      }
    }
  }
}
// 使用getline 方法
char* aker_getLine1() {
  size_t size = 0;
  char *line = NULL;
  printf("aker>");
  if (getline(&line, &size, stdin) < 1) {
    if (feof(stdin)) {
      printf("退出中...\n");
      printf("欢迎下次使用\n");
      exit(EXIT_SUCCESS);
    } else {
      perror("aker");
      exit(EXIT_FAILURE);
    }
  }
  return line;
}
// 使用readline 方法
// gcc -std=c99 shell5.c -ledit -o shell5
char* aker_getLine2() {
  char* line = readline("aker>");
  if (!line) {
    printf("退出中...\n");
    printf("欢迎下次使用\n");
    exit(EXIT_SUCCESS);
  }
  return line;
}

void aker_process() {
  char *line;
  char **args;
  int status;
  do {
    line = aker_getLine();
    args = aker_getArgs(line);
    status = aker_exec(args);
    add_history(line);
    free(args);
    free(line);
  } while (status);

  puts("end!");
}

int main() {
  aker_process();
}
```

## 学习资料

[学习来源](https://brennan.io/2015/01/16/write-a-shell-in-c/)
