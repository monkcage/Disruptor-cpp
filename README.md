# Disruptor-cpp

[![Build Status](https://travis-ci.org/Abc-Arbitrage/Disruptor-cpp.svg?branch=master)](https://travis-ci.org/Abc-Arbitrage/Disruptor-cpp)
[![Build status](https://ci.appveyor.com/api/projects/status/7mg15p3d7n8jqjmg?svg=true)](https://ci.appveyor.com/project/Abc-Arbitrage/disruptor-cpp)


## Overview

Disruptor-cpp is a fully functional C++ port of the [LMAX disruptor](https://lmax-exchange.github.io/disruptor/). 
Implements all the features available in java Disruptor v3.3.7.

## Building

### Compilers

* Clang 3.8 or newer
* GCC 5.0 or newer
* Microsoft Visual C++ 2015 or newer

### Linux

```sh
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=release
make
```

Optionally you may want to compile and run the unit tests and benchmarks. The tests compilation is activated by means of **DISRUPTOR_BUILD_TESTS** flag: 

```sh
cmake .. -DCMAKE_BUILD_TYPE=release -DDISRUPTOR_BUILD_TESTS=true
```

### Windows 

The simplest way to compile the library on Windows is to use the provided Visual Studio solution files:
* Disruptor-lib.sln - the solution only includes the library.
* Disruptor-all.sln - the solution includes the library, benchmarks and unit tests.

## Getting started

To give you a taste of C++ disruptor let us consider a very basic example where the event is passed from producer to consumer. The event will carry a single long value:  

```Cpp
struct LongEvent
{
    long value;
};
```

The consumer will print the event value to standard output and also notify the publishing thread when everything is processed:

```Cpp
struct PrintingEventHandler : Disruptor::IEventHandler< LongEvent >
{
    explicit PrintingEventHandler(int toProcess) : m_actuallyProcessed(0), m_toProcess(toProcess)
    {}

    void onEvent(LongEvent& event, int64_t, bool) override
    {
        std::cout << "Event: " << event.value << std::endl;

        if (++m_actuallyProcessed == m_toProcess)
            m_allDone.notify_all();
    }

    void waitEndOfProcessing()
    {
        std::unique_lock<decltype(m_mutex)> lk(m_mutex);
        m_allDone.wait(lk);
    }

private:
    std::mutex m_mutex;
    std::condition_variable m_allDone;
    int m_toProcess;
    int m_actuallyProcessed;
};
```

Now we can wire all the things together:

```Cpp
    auto const ExpectedNumberOfEvents = 10000;
    auto const RingBufferSize = 1024;

    // Instantiate and start the disruptor
    auto eventFactory = []() { return LongEvent(); };
    auto taskScheduler = std::make_shared< Disruptor::ThreadPerTaskScheduler >();
    
    auto disruptor = std::make_shared< Disruptor::disruptor<LongEvent> >(eventFactory, RingBufferSize, taskScheduler);
    auto printingEventHandler = std::make_shared< PrintingEventHandler >(ExpectedNumberOfEvents);

    disruptor->handleEventsWith(printingEventHandler);

    taskScheduler->start();
    disruptor->start();

    // Publish events
    auto ringBuffer = disruptor->ringBuffer();
    for (auto i = 0; i<ExpectedNumberOfEvents; ++i)
    {
        auto nextSequence = ringBuffer->next();
        (*ringBuffer)[nextSequence].value = i;
        ringBuffer->publish(nextSequence);
    }

    // Wait for the end of execution and shutdown
    printingEventHandler->waitEndOfProcessing();

    disruptor->shutdown();
    taskScheduler->stop();
```

For more details, please refer the original [Java Disruptor documentation](https://lmax-exchange.github.io/disruptor).
