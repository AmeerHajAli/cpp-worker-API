# cpp-worker-API
# High order APIs

This document will give a brief introduction of high order APIs of c++ worker, however i think Python and Java should also provide similar APIs.

These are the APIs:

# async

## thread async

**API**
```
template<typename F, typename... Args>
auto async(F&& f, Args&&... args)-> boost::unique_future<decltype(f(args...))>
```

**Description**

lauch an async operation with arbitrary arguments in a new thread, return a future object; The future object is a  continuable， can be continued with the 'then' API.

## policy async
```
enum launch{
    async = 1,     //in new thread
    deferred = 2,  //in the same thead
    executor = 4,  //in thread pool
};

template<typename F, typename... Args>
auto async(launch policy, F&& f, Args&&... args)-> boost::unique_future<decltype(f(args...))>
```

**Description**

Policy-based lauch an async operation with arbitrary arguments in a new thread, return a future object; The future object is a  continuable， can be continued with the 'then' API.

## thread_pool async

**API**
```
template<typename Executor, typename F, typename... Args>
auto async(Executor& ex, F&& f, Args&&... args)-> boost::unique_future<decltype(f(args...))>
```

**Description**

lauch an async operation with arbitrary arguments in a given thread_pool, return a future object;

# then

## non-policy then

**API**
```
template<typename F>
auto future<R>::then(F&& func) -> boost::unique_future<typename boost::result_of<F(boost::unique_future)>::type>;
```

**Description**

A member function of future, used to continue the future in a chain. lauch an async operation with parant future as argument, return a future object.

## policy then

**API**
```
enum launch{
    async = 1,     //in new thread
    deferred = 2,  //in the same thead
    executor = 4,  //in thread pool
};

template<typename F>
auto future<R>::then(launch policy, F&& func) -> boost::unique_future<typename boost::result_of<F(boost::unique_future)>::type>;
```
**Description**

A member function of future, used to continue the future in a chain. Policy-based lauch an async operation with parant future as argument, return a future object.

# when_all

## variadic template when_all
**API**
```
template<typename... F>
auto when_all(F... fns);
```
**Description**

Combine arbitrary futures, return a future object which will wait for all the futures finished.

## container when_all
**API**
```
template<typename InputIterator>
auto when_all(InputIterator first, InputIterator end);
```
**Description**

Combine many futures from a container, such as vector/list, return a future object which will wait for all the futures finished.

# when_any

similar with when_all, when_any return a future object will wait for any of the futures.

# quick example

```
TEST(RayApiTest, ThenTest){
    auto future = async(Plus1, 2)
            .then([](Future<int> f){
                return f.get();
            })
            .then([](Future<int> f){
                return f.get() + async(Plus, 2, 3).get();
            });

    EXPECT_EQ(8, future.get());
}

TEST(RayApiTest, ThenExecutorTest){
    thread_pool ex;
    auto future = async(ex, Plus1, 2)
            .then(ex, [](Future<int> f){
                return f.get();
            })
            .then(ex, [](Future<int> f){
                return f.get() + async(Plus, 2, 3).get();
            });

    EXPECT_EQ(8, future.get());
}

TEST(RayApiTest, ThenWaitTest){
    auto future = async(Plus1, 2)
            .then([](Future<int> f){
                return f.get();
            })
            .then([](Future<int> f){
                return f.get() + async(Plus, 2, 3).get();
            });

    auto status = future.wait_for(boost::chrono::milliseconds(1000));
    if(status==boost::future_status::ready){
        EXPECT_EQ(8, future.get());
    }
}

TEST(RayApiTest, WhenAllVarTest) {
    auto f = boost::when_all(async(Return1), async(Plus1, 3), async(Plus, 2, 3));

    auto future = f.then([](decltype(f) fu) {
        auto t = fu.get();
        EXPECT_TRUE(std::get<0>(t).is_ready());
        EXPECT_TRUE(std::get<1>(t).is_ready());
        EXPECT_TRUE(std::get<2>(t).is_ready());
    });

    future.get();
}

TEST(RayApiTest, WhenAllTest) {
    std::vector<boost::unique_future<int>> v;
    v.push_back(async(Return1));
    v.push_back(async(Plus1, 3));
    v.push_back(async(Plus, 2, 3));

    auto f = boost::when_all(v.begin(), v.end());

    auto f1 = f.then([](decltype(f) fu) {
        auto v = fu.get();
        for (auto& t : v) {
            EXPECT_TRUE(t.is_ready());
        }
    });

    f1.get();
}

TEST(RayApiTest, WhenAnyTest) {
    std::vector<boost::unique_future<int>> v;
    v.push_back(async(Return1));
    v.push_back(async(Plus1, 3));
    v.push_back(async(Plus, 2, 3));

    auto f = boost::when_any(v.begin(), v.end());

    auto future = f.then([](decltype(f) fu) {
        auto v = fu.get();
        bool r = false;
        for (auto& t : v) {
            r|=t.is_ready();
        }

        EXPECT_TRUE(r);
    });

    future.get();
}
```
