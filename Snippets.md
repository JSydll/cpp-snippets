# Usefull C++ Snippets
[1 Modern C++ best practices](#bp)

[1.1 Introducing abstraction](#bp-abstract)

[1.1.1 For_each to encapsulate functionality](#bp-abstract-foreach)

[2 Implementing design pattern](#pattern)

[3 Integrating C in C++](#ccpp)

[3.1 Wrapper for C-style strings](#ccpp-cstring-wrapper)

[4 Utilities](#util)

[4.1 Timing](#util-time)

[4.1.1 Get system date & time string](#util-time-string)

[4.1.2 Get milliseconds since specific timepoint](#util-time-ms)

[4.2 Filesystem operations](#util-fs)

[4.2.1 Check for existence](#util-fs-existence)

[4.2.2 Check for emptiness](#util-fs-empty)

[4.2.3 RAII text file writer](#util-fs-filewriter)

[4.3 Network Monitoring (Linux)](#util-net)

[4.3.1 Checking and waiting for a network connection](#util-net-check)

[4.3.2 Network throughput tracking](#util-net-track)

[5 Dedicated tasks](#tasks)

[5.1 Finding a node in a tree](#task-tree)

---

## 1 Modern C++ best practices <a name="bp"/>

### 1.1 Introducing abstraction <a name="bp-abstract"/>

#### 1.1.1 For_each to encapsulate functionality <a name="bp-abstract-foreach"/>
```cpp
#include <algorithm>

namespace ranges {
    template<typename range_t, typename func_t>
    func_t for_each(range_t& range, func_t f){
        return std::for_each(begin(range), end(range), f);
    }
}
```

Note: This is meant to encapsulate the inner workings of f and avoids the overhead of writing *begin(...), end(...)* on every call of *for_each*. The function f takes one parameter of the range item type. If the implementation of f should be visible inline, use range-based for loop instead or *for_each* with a lambda expression. 

## 2 Implementing design pattern <a name="pattern"/>

## 3 Integrating C in C++ <a name="ccpp"/>

### 3.1 Wrapper for C-style strings <a name="ccpp-cstring-wrapper"/>
```cpp
#include <string>
#include <vector>

class cStr {
public:
	/** Convert a string into a char vector on construction.
	 * The vector container is a convenient way to enable access to char*'s while leaving the 
	 * responsibility for resource allocation and cleanup to the well defined STL.
	 * @param str String to be accessible as char*. */
	cStr(std::string str) : _vec(str.begin(), str.end()){
		this->_vec.push_back('\0');
	}
	~cStr(){ };
	/** Getter for the char* of a C-style string */
	char* get(){
		return &this->_vec[0];
	};
private:
	std::vector<char> _vec;
};
```

## 4 Utilities <a name="util"/>

### 4.1 Timing <a name="util-time"/>

#### 4.1.1 Get system date & time string <a name="util-time-string"/>
```cpp
#include <chrono>
#include <ctime>
#include <sstream>
#include <iomanip>
#include <string>

/** Format string in the form of "%Y-%m-%d %X" */
std::string getSystemTime(std::string format){
    auto now = std::chrono::system_clock::now();
    auto in_time_t = std::chrono::system_clock::to_time_t(now);
    std::stringstream ss;
    ss << std::put_time(std::localtime(&in_time_t), format.c_str());
    return ss.str();
}
```

#### 4.1.2 Get milliseconds since specific timepoint <a name="util-time-ms"/>
```cpp
#include <chrono>

/** Get the chrono::time_point of 01-01-2018 0:00:00 */
std::tm then_tm{ 0, 0, 0, 1, 0, 118 };
auto then = std::chrono::system_clock::from_time_t(std::mktime(&then_tm));
auto now = std::chrono::system_clock::now();
unsigned long long ms = std::chrono::duration_cast<std::chrono::milliseconds>(now - then).count();
```

### 4.2 Filesystem operations <a name="util-fs"/>

#### 4.2.1 Check for existence <a name="util-fs-existence"/>
```cpp
#include <sys/stat.h>
#include <sys/types.h>

bool isValidFile(std::string path){
    struct stat sb;
    stat(path.c_str(), &sb);
    return ( sb.st_mode & S_IFREG );
}
bool isValidDirectory(std::string path){
    struct stat sb;
    stat(path.c_str(), &sb);
    return ( sb.st_mode & S_IFDIR );
}
```

The above gives more options for specifying the file/directory type. If only the *access* to a file is of interest, a more lightweight solution can be employed:

```cpp
#include <unistd.h>

bool isAccessibleFile(std::string path){
    return ( std::access( path.c_str(), F_OK ) != -1 );
}
```

#### 4.2.2 Check for emptiness <a name="util-fs-empty"/>
```cpp
#include <sys/stat.h>
#include <sys/types.h>
#include <dirent.h>

bool isEmptyDirectory(std::string path){
    bool empty = true;
    /** Insert checking for existence here...*/
    DIR* directory = opendir( path.c_str() );
    struct dirent* singleFile;
    while( (singleFile = readdir(directory)) != NULL ){
        if(!strcmp(singleFile->d_name, ".") || !strcmp(singleFile->d_name, "..")){
	    empty = false;
	    break;
	}
    }
    closedir(directory);
    return empty;
}
bool isEmptyFile(std::string path){
    bool empty = false;
    /** Insert checking for existence here...*/
    std::ifstream thisFile(path);
    if(thisFile)
	empty = (thisFile.peek() == std::ifstream::traits_type::eof());
    thisFile.close();
    return empty;
}
```

#### 4.2.3 RAII text file writer <a name="util-fs-filewriter"/>
```cpp
#include <iostream>
#include <fstream>

/** Will create new file if not found. */
class FileWriter {
public:
    FileWriter(std::string path, bool append){
        m_filestream.open(path, (append ? std::ios::app : std::ios::out) );
    }
    ~FileWriter(){
        m_filestream.close();
    }
    template <class T>
    FileWriter& operator<<(const T& content){
        m_filestream << content;
        return *this;
    }
private:
    std::ofstream m_filestream;
};
```

### 4.3 Network Monitoring (Linux) <a name="util-net"/>

#### 4.3.1 Checking and waiting for a network connection <a name="util-net-check"/>
```cpp
#include <string>
#include <array>
#include <unistd.h>
#include <memory>

bool ready = false;
unsigned short waited = 0;
std::array<char, 10> buff;
std::string cmd = "ifconfig ppp0 2>/dev/null | grep -c 'inet addr'";
std::string result;
do {
    // Read the result of the ifconfig command
    std::shared_ptr<FILE> pipe(popen(cmd.c_str(), "r"), pclose);
    if(!pipe)
	/** Error handling goes here...*/
    while(!feof(pipe.get())){
	if(fgets(buff.data(), 10, pipe.get()) != nullptr)
	    result += buff.data();
    }
    // Check if the network connection is established
    if(result.find("1") != std::string::npos){
	ready = true;
	break;
    } else {
	// If not, wait 20 secs and try again.
	std::this_thread::sleep_for(std::chrono::seconds(20));
	waited += 20;
    }
    // Try until the maximum amount of time passed is reached.
    } while(waited <= SystemConstants::FW_MAX_NET_WAIT);
```

#### 4.3.2 Network throughput tracking <a name="util-net-track"/>
```cpp
#include <string>
#include <cstdlib>
#include <unistd.h>
#include <memory>
#include <regex>

unsigned long RXbytes = 0;
unsigned long TXbytes = 0;
std::array<char, 200> buff;
std::string result;
/** Issue a system command for the ifconfig, grep the info */
std::string cmd = "ifconfig ppp0 | grep 'RX bytes' | sed -e 's/^\\s*//'";
std::shared_ptr<FILE> pipe(popen(cmd.c_str(), "r"), pclose);
if(!pipe)
    /** Error Handling goes here... */
while(!feof(pipe.get())){
    if(fgets(buff.data(), 200, pipe.get()) != nullptr)
	result += buff.data();
}
/** Using regex to find the values for RX/TX */
std::regex rxVal(".*(RX bytes:)([0-9]+)\\s.*");
std::regex txVal(".*(TX bytes:)([0-9]+)\\s.*");
std::smatch matches;
if(std::regex_search(result, matches, rxVal) && matches.size() > 1)
    RXbytes = std::stoul(matches.str(2));
if(std::regex_search(result, matches, txVal) && matches.size() > 1)
    TXbytes = std::stoul(matches.str(2));
```

## 5 Dedicated tasks <a name="tasks"/>

### 5.1 Finding a node in a tree <a name="task-tree"/>

Assuming a tree with nodes of type *node_t* and a property *children* of type *std::vector<std::shared_ptr<node_t>>* like
```
root
  |-- child
  |-- child
  |     |-- child
  ...   ...
```
a recursive algorithm can be implemented as follows:
```cpp
#include <memory>
#include <vector>

std::shared_ptr<node_t> findNodeByName(const std::shared_ptr<node_t>& root, std::string name){
    shared_ptr<node_t> childMatch;
    if(root->name == name){
        return root;
    } else {
        if(root->children.size()){
            for(const auto& child : root->children){
                childMatch = findNodeByName(child, name);
                if(childMatch != nullptr) break;
            }
            return childMatch;
        } else {
            return nullptr;
        }
    }
}
```
