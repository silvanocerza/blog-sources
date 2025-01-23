---
title: A simple way to run only one process per application in C++
date: 2019-04-10T17:51:49+02:00
tags:
  - cpp
---

Some times it might be necessary to limit the number of processes running at the same time for a certain application. There might be several reasons for this, to prevent data corruption for example. This is a simple cross-platform way to do it.

---

We're going to use C++ 17 and the new [filesystem library](https://en.cppreference.com/w/cpp/filesystem), this will permit us to minimize the use of `ifdef` since we won't have to use the POSIX or Windows APIs that much.

Our solution will work roughly like this:

1. On launch read the lock file to check if there are other processes running
2. Write the current process ID into the lock file
3. On close remove the current process ID from the lock file

In case our applications crashes obviously it won't be able to remove its process ID from the lock file but the solution we're going to use won't have to worry about that.

We'll get the includes out of the way right now, so if you want to try the code after each step you'll have no issue. These all are you need:

```cpp
#include <algorithm>
#include <filesystem>
#include <fstream>
#include <string>
#include <vector>

#ifdef _MSC_VER
#include <Windows.h>
#include <Psapi.h>
#include <tchar.h>
#else
#include <unistd.h>
#endif
```

### Lock File & Process ID

First of all we'll need to reliably retrieve the path of our lock file, and keep using that all the time. This is where we'll use the new filesystem library, luckily it offers us a function that returns a path to the OS temporary directory. We can name our file however we want, but it's important that its name is unique and doesn't change at runtime.

```cpp
std::string lockFilePath()
{
    static std::string file = std::filesystem::temp_directory_path().string() + "/MyLockFile";
    return file;
}
```

As you might have noticed we declared our `file` variable as `static`, this will initialize the variable only the first time that piece of code is executed and never do it again during the life time of the program. You can find more information on static local variables on the [official documentation](https://en.cppreference.com/w/cpp/language/storage_duration#Static_local_variables).

Now we need to know our current process ID, to do that we must use the OS APIs and is one of the few spots where we have to use `ifdef`, on Windows we'll use the PSAPI, on the other platforms we use POSIX.

```cpp
int processId()
{
#ifdef _MSC_VER
    return GetCurrentProcessId();
#else
    return ::getpid();
#endif
}
```

There's nothing magical about this function, it returns the process ID and that's it.

### Lock and Unlock

The two previous functions are enough to handle the lock and unlock steps.
The locking is pretty straightforward, open the file, append the current process ID and close the file.

```cpp
// Writes current app PID to lock file
void lockProcess()
{
    std::fstream lockFile(lockFilePath(), std::ios::out | std::ios::app);
    lockFile << processId() << std::endl;
    lockFile.close();
}
```

The unlock is a bit more complicated instead; it has to open the lock file, read all the process IDs, remove the current process ID and write back the others.

```cpp
// Removes current PID from lock file
void unlockProcess()
{
    std::fstream lockFile;
    lockFile.open(lockFilePath(), std::ios::in);

    std::vector<std::string> ids;
    std::string id;
    while (std::getline(lockFile, id)) {
        if (std::stoi(id) != processId()) {
            ids.push_back(id);
        }
    }
    lockFile.close();
    lockFile.open(lockFilePath(), std::ios::out | std::ios::trunc);
    for (const auto& id : ids) {
        lockFile << id << std::endl;
    }
    lockFile.close();
}
```

This are the building blocks to handle our lock file, but we need to put it to use or it would be kind of pointless to just save the running processes of our application.

### Am I Alone?

Our goal is to verify if there are other processes of our application running so we'll use the lock file to do just that. Our `isOnlyInstance` function will return whether there is another process running by searching in the currently running processes if there is one with the same ID as the ones in the lock file.

```cpp
// Returns wether there is another instance of the app with different PID running
bool isOnlyInstance()
{
    std::fstream lockFile(lockFilePath(), std::ios::in);
    std::vector<std::string> ids;
    std::string id;
    while (std::getline(lockFile, id)) {
        ids.push_back(id);
    }
    lockFile.close();

    auto procs = processList();
    for (auto id : ids) {
        if (std::find(procs.cbegin(), procs.cend(), id) != procs.cend()) {
            return false;
        }
    }
    return true;
}
```

Now we need to implement our `processList` function, as its name clearly says it returns a list of running processes. This is another spot for `ifdefs`.

On Windows we're using the [EnumProcesses](https://docs.microsoft.com/en-us/windows/desktop/api/Psapi/nf-psapi-enumprocesses) function from the PSAPI, it expects a `DWORD[]` that will be filled with the running process IDs, the size of that array and a `LPDWORD`, that is a `DWORD*`, that will contain the number of bytes returned in the processes array. A `DWORD` is a Windows `typedef` for `unsigned long`. We then iterate the array of IDs and return it as a vector of strings.

Instead on other platforms we take advantage of `/proc`, that contains a folder named after each running process ID, who themselves will contain the informations of that process. We won't need more informations other the IDs so we just iterate the directories in `/proc`, push into our vector and return.

Here we're using again the `filesystem` library, notice how convenient it is to iterate a directory content using a [directory_iterator](https://en.cppreference.com/w/cpp/filesystem/directory_iterator). `p` in this case is a [directory_entry](https://en.cppreference.com/w/cpp/filesystem/directory_entry) that we convert to [path](https://en.cppreference.com/w/cpp/filesystem/path) to retrieve the directory name, the process ID in this case.

```cpp
std::vector<std::string> processList()
{
#ifdef _MSC_VER
    DWORD aProcesses[1024], cbNeeded;

    // Returns zero on failure but we ignore because we're brave enough
    EnumProcesses(aProcesses, sizeof(aProcesses), &cbNeeded);

    DWORD cProcesses = cbNeeded / sizeof(DWORD);

    std::vector<std::string> result;
    for (DWORD i = 0; i < cProcesses; i++) {
        result.push_back(std::to_string(aProcesses[i]));
    }
    return result;
#else
    std::vector<std::string> processes;
    for (const auto& p : std::filesystem::directory_iterator("/proc")) {
        processes.push_back(p.path().filename());
    }
    return processes;
#endif
}
```

### Usage

Now that we have all the pieces we can put our mini library to use.

```cpp
int main()
{
    std::cout << processId() << std::endl;
    if (!isOnlyInstance()) {
        std::cout << "Another process is running" << std::endl;
    }
    lockProcess();

#ifdef _MSC_VER
	Sleep(5000);
#else
    ::sleep(5);
#endif
    unlockProcess();
    return 0;
}
```

This is just a minimal example, it prints the process ID and notifies you if another process is running, then locks the current one and waits 5 seconds before unlocking and terminating.

What about crashes though?

### Crash and Burn!

The code as it is fine and working, we could stop here and call it a day. The problem is that in case of crashes some IDs might be stuck in our lock file until the next reboot, if the user rarely reboots and the app crashes several times --- I hope it doesn't --- our lock file might grow a bit. This should be rare but nonetheless we can handle that by cleaning the lock file ourselves.

Our cleanup function will read all the IDs from the lock file and rewrite it removing those not currently running. To use it we can just call it atop our `main` and that's it, but it's not strictly necessary.

```cpp
// Removes PIDs from lock file that are not running anymore, this mainly cleanups after
// crashes since unlockProcess would not be called
void cleanLockFile()
{
    std::fstream lockFile;
    lockFile.open(lockFilePath(), std::ios::in);
    std::vector<std::string> ids;
    std::string id;
    while (std::getline(lockFile, id)) {
        ids.push_back(id);
    }
    lockFile.close();

    lockFile.open(lockFilePath(), std::ios::out | std::ios::trunc);
    auto procs = processList();
    for (auto id : ids) {
        if (std::find(procs.cbegin(), procs.cend(), id) != procs.cend()) {
            lockFile << id << std::endl;
        }
    }
    lockFile.close();
}
```

### Final thoughts

Now we're pretty much done, that's nothing much to do here, you can find an example project on [Github](https://github.com/silvanocerza/locker). Feel free to use it for your project, the license is pretty permissive.

Some of you might ask why we went through all the trouble of saving our process IDs to a file instead of searching for a process with the same name, that wouldn't be the most efficient way though. Neither Windows' PSAPI nor POSIX offer a way to get a list of processes given its name, so we should have iterated all processes every time we open our application to know if there was another one running. Using the lock file mechanism we know right away if another instance is running, if it's empty just let it be, instead if there is one or more IDs we verify if at least one of them is the list of running process IDs.

Thank you for reading this trough the end, I hope you enjoyed. If you have anything to say feel free to leave a comment.
