---
title: boost
tags:
  - C++
toc: true
date: 2019-03-01 16:11:37
---

# boost::thread_specific_ptr

``` cpp
#include <boost/thread/thread.hpp>
#include <boost/thread/mutex.hpp>
#include <boost/thread/tss.hpp>
#include <iostream>

boost::mutex io_mutex;
boost::thread_specific_ptr<int> ptr([](int *ptr) {
    std::cout << "delete ptr" << std::endl;
    delete ptr;
});

class count
{
public:
    count(int id) : id(id)
    {
    }

    void operator()()
    {
        if(ptr.get() == nullptr)
        {
            ptr.reset(new int(0));
        }

        for(int i = 0; i < 10; ++i)
        {
            (*ptr)++;
            boost::mutex::scoped_lock lock(io_mutex);
            std::cout << id << " : " << *ptr << std::endl;
        }
    }

private:
    int id;
};

int main(int argc, char* argv[])
{
    boost::thread thrd1(count(1));
    boost::thread thrd2(count(2));
    thrd1.join();
    thrd2.join();
    return 0;
}
```



<!--more-->