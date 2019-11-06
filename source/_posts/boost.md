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
# std::shared_ptr
``` cpp
#include <string>
#include <iostream>
#include <memory>
#include <vld.h>

class Person
{
public:
    Person()
    {
        std::cout << "person construct" << std::endl;
    }

    ~Person()
    {
        std::cout << "~person destruct" << std::endl;
    }

public:
    int age;
    std::string name;
};


class Client
{
public:
    Client()
    {
        std::cout << "client construct" << std::endl;
    }

    ~Client()
    {
        std::cout << "~ client destruct" << std::endl;
    }

public:
    std::shared_ptr<Person> m_person;
};

class NetClient
{
public:
    NetClient()
    {
        std::cout << "net client construct, use count: " << client.use_count() << std::endl;
    }

    ~NetClient()
    {
        client.reset();
        std::cout << "net client destruct, use count: " << client.use_count() << std::endl;
    }

    void function()
    {
        const int size = 10;

        client = std::make_shared<Client>();
        std::cout << "client use count: " << client.use_count() << std::endl;

        client->m_person.reset(new Person[size], [](Person *p) { delete[] p; });

        std::cout << "swap" << std::endl;
        std::shared_ptr<Client> c2;
        std::cout << "c2 use count: " << c2.use_count() << std::endl;

        //client.swap(c2);

        c2 = client;

        std::cout << "c2 use count: " << c2.use_count() << std::endl;
        std::cout << "client use count: " << client.use_count() << std::endl;

        c2.reset();

        std::cout << "c2 use count: " << c2.use_count() << std::endl;
        std::cout << "client use count: " << client.use_count() << std::endl;
    }

private:
    std::shared_ptr<Client> client;
};

int main()
{
    NetClient net;

    net.function();

    return 0;
}

```