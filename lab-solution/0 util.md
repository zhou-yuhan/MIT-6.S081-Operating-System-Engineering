# Lab Util

1. sleep

```cpp
#include "kernel/stat.h"
#include "kernel/types.h"
#include "user/user.h"


int main(int argc, char* argv[]) {
    if (argc != 2) {
        fprintf(2, "Usage: sleep [seconds]\n");
        exit(1);
    }
    int n = atoi(argv[1]);
    sleep(n);
    exit(0);
}
```

2. pingpong

```cpp

#include "kernel/stat.h"
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char* argv[]) {
    char buf[5];
    int fd[2] = {3, 4};
    pipe(fd);
    int pid = fork();
    if (pid > 0) {
        write(fd[0], "A", 1);
        wait((int*)0);
        read(fd[0], buf, 1);
        fprintf(1, "%d: received pong\n", getpid());
        exit(0);
    } else {
        read(fd[1], buf, 1);
        fprintf(1, "%d: received ping\n", getpid());
        write(fd[1], buf, 1);
        exit(0);
    }
}
```

3. primes

```cpp

#include "kernel/stat.h"
#include "kernel/types.h"
#include "user/user.h"

void creat_proc(int p[2]) {
    int prime, next;
    close(p[1]);

    if(read(p[0], &prime, sizeof(int)) != sizeof(int)) {
        fprintf(2, "child process failed to read\n");
        exit(1);
    } 

    fprintf(1, "prime %d\n", prime);

    if(read(p[0], &next, sizeof(int)) == sizeof(int)) {
        int pp[2];
        pipe(pp);
        if(fork() == 0) {
            creat_proc(pp);
        } else {
            close(pp[0]);
            if(next % prime != 0) {
                write(pp[1], &next, sizeof(int));
            } 

            while(read(p[0], &next, sizeof(int))) {
                if(next % prime != 0)
                    write(pp[1], &next, sizeof(int));
            }

            close(p[0]);
            close(pp[1]);
            wait((int*) 0);
            exit(0);
        }
    }
    exit(0);
}

int main(int argc, char* argv[]) {
    int p[2];
    pipe(p);

    if(fork() == 0) {
        creat_proc(p);
    } else {
        close(p[0]);
        for(int i = 2; i <= 35; ++i) {
            write(p[1], &i, sizeof(int));
        }
        close(p[1]);
        wait((int*) 0);
    }
    exit(0);
}
```

4. find

```cpp

#include "kernel/fs.h"
#include "kernel/stat.h"
#include "kernel/types.h"
#include "user/user.h"

void find(char* path, char* target) {
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, 0)) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }
    if (st.type == T_DIR) {
        strcpy(buf, path);
        p = buf + strlen(buf);
        *p++ = '/';
        while (read(fd, &de, sizeof(de)) == sizeof(de)) {
            if (de.inum == 0 || strcmp(de.name, ".") == 0 ||
                strcmp(de.name, "..") == 0)
                continue;
            memmove(p, de.name, DIRSIZ);
            p[DIRSIZ] = 0;
            find(buf, target);
        }
    }
    close(fd);

    for (p = path + strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;
    if (strcmp(p, target) == 0) {
        path[strlen(path)] = '\n';
        fprintf(1, path);
    }
    return;
}

int main(int argc, char* argv[]) {
    if (argc != 3) {
        fprintf(1, "Usage: find [dir] [file]\n");
        exit(1);
    }
    char* dir = argv[1];
    char* file = argv[2];
    find(dir, file);
    exit(0);
}
```

5. xargs

```cpp

#include "kernel/fs.h"
#include "kernel/stat.h"
#include "kernel/types.h"
#include "user/user.h"

void find(char* path, char* target) {
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, 0)) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }
    if (st.type == T_DIR) {
        strcpy(buf, path);
        p = buf + strlen(buf);
        *p++ = '/';
        while (read(fd, &de, sizeof(de)) == sizeof(de)) {
            if (de.inum == 0 || strcmp(de.name, ".") == 0 ||
                strcmp(de.name, "..") == 0)
                continue;
            memmove(p, de.name, DIRSIZ);
            p[DIRSIZ] = 0;
            find(buf, target);
        }
    }
    close(fd);

    for (p = path + strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;
    if (strcmp(p, target) == 0) {
        path[strlen(path)] = '\n';
        fprintf(1, path);
    }
    return;
}

int main(int argc, char* argv[]) {
    if (argc != 3) {
        fprintf(1, "Usage: find [dir] [file]\n");
        exit(1);
    }
    char* dir = argv[1];
    char* file = argv[2];
    find(dir, file);
    exit(0);
}
```