---
title: Constexpr c++{14} and metprogramming
categories: cpp
layout: post
---

One guy said 'if there were award for the most confusing word in C++11, constexpr would probably win it.', and he was be correct. So what the difference between `constexpr` and regular `const`? `constexpr` can be used in 2 cases - apply `constexpr` to object and apply `constexpr` to function. 
Let's look both cases:

**Apply to object**

```
constexpr auto size = 10;
std::array<float, size> arr;
```
Your object will be assigned by constant calculated expression in a **compile time**. It's mean your variable size is just regular C++98 `const`, and you can use this variable for like initialization static arrays.

**Apply to function**

```
constexpr auto DEBUG_VERSION = false; //constexpr object
constexpr auto calculateArrayLength() //C++14
{
		if(DEBUG_VERSION){
				return 20 * 30 - 70 * 10;
		}else{
				return 10;
		}
}
constexpr auto calculateArrayLength() //C++11
{
		//C++ doesnt support any other expressions inside constexpr function
		return DEBUG_VERSION? 20 * 30 - 70 * 10: 10;
}
...
constexpr auto stack_size = calculateArrayLength();
const std::array<float,stack_size> stack_area;
...
```
As we can see - `constexpr` indicate which function evaluation result can be calculated in compilation time. But using `const` + class function means we dont change any member variable of the class, as we can see below.

```
class String{
    using str_container = std::vector<char>;
private:
    str_container data;
public:
    String(){}
    String(std::string data){

        this->data.resize(data.length() + 1);
        std::copy(data.c_str(), data.c_str() + data.length() + 1, this->data.begin());
        this->data.resize(data.length());//remove '\0' last character
    }
    String(const str_container& data):data(data){}

    String operator+(const String& rhs) const{ //We use imutable string way, so we do not change internal state (we add keyword const at end of function)
        str_container data_container(this->data.size() + rhs.data.size() + 1);
        std::copy(this->data.begin(),this->data.end(), data_container.begin());
        std::copy(rhs.data.begin(), rhs.data.end(), data_container.begin() + this->data.size());
        data_container.resize(data_container.size() - 1);//Resize
        return String(data_container);
    }
};

```
As we can see `constexpr` always `const`, but `const` not always `constexpr`.

**constexpr Class functions**

Yes, we can create constexpr class instances. 

```
#include <array>
#include <iostream>
//1x2 Matrix
template <typename DataType>
class Vector2 {
public:
public:
    constexpr Vector2(){
    }
    constexpr Vector2(DataType x, DataType y):x(x), y(y){
    }

    DataType x;
    DataType y;
};

#define Vector2f Vector2<float>
using Vector2d = Vector2<double>; //Walk away old C++98 define!
using Vector2i = Vector2<int>;
using Vector2l = Vector2<long>;

//1x3 Matrix
template <typename DataType>
class Vector3: public Vector2<DataType>{
public:
    DataType z;
    constexpr Vector3(){

    }
    constexpr Vector3(DataType x, DataType y, DataType z): Vector2<DataType>(x,y), z(z){

    }
		// 3 keywords which containt 'const' keyword inside, hah
    constexpr auto operator+(const Vector3<DataType>& rhs) const{    
        return Vector3<DataType>(this->x + rhs.x, this->y + rhs.y, this->z + rhs.z);
    }
    constexpr auto max() const{
        return std::max(std::max(this->x,this->y), this->z);
    }
};
using Vector3f = Vector3<float>;
using Vector3d = Vector3<double>;
using Vector3i = Vector3<int>;
using Vector3l = Vector3<long>;

int main(){
    constexpr Vector3i v(10,20,30);

    constexpr Vector3i v1(20,30,40);
    constexpr auto v3 = v + v1;

    const std::array<int, v3.max()> arr = {};// array size = 70
    std::cout<<arr.size()<<std::endl;  //Will print out 70

    return EXIT_SUCCESS;
}

```
Generating `constexpr` Vector3i class instances, calculating the sum of 2 vectors and pass the max value in array initialization as the array size. At function operator+ we can see 3 `const` keywords, first one indicate `constexpr` return value from a function, the second one - mean we send constant object inside the function and last one - mean this function do not change the internal state of the class. Also you can `constexpr` functions as regular functions, like `auto vec = Vector3i(0,0,0);`

**Summary**

`constexpr` can be applied to variables, and functions. When you apply `constexpr` to the variable you indicate - this variable will be assigned by a constant expression. In another hand when we apply `constexpr` to function - we indicate this function **can be** used as a constant expression.

----

But what about **metaprograming**?
In a few words [metaprograming](https://en.wikipedia.org/wiki/Metaprogramming) - is a programming technique in which program manipulate programs, the program can read, edit, generate, analyze another program. Also, I like this definition **"write code then writes code"**.  If you familiar with Java, C# or any other language which have Reflection, reflection is a form metaprogramming. Also, you can write some program which generates source code, and after that, you compile this code - and you have a working program which was be created by another program. 

Aas we can see `constexpr` values, will be calculated at compilation time, as the case we can implement some algorithm which does no depends on input data, and evaluate them inside our program for **zero** time, by zero I mean constant O(1) complexity. Let's try to implement something, simple like Fibonacci number. 

The sequence `F(n)` of fibonacci number is defined by recurrence relation:
`F(n) = F(n-1) + F(n-2);
F(0) = 1;F(1) = 1;
`
`
F(0) = 1;F(1) = 1; F(2) = 1; F(3) = F(2) + F(1); F(4) = F(3) + F(2); F(n) = F(n-1) + F(n-2)
`
And of course, Fibonacci number has [Closed-form expression](https://en.wikipedia.org/wiki/Fibonacci_number#Closed-form_expression).
The first thing that catches your eye is this expression has really bad complexity if we decide to use this equation without any optimization like memoization. In current expression, we compute the same functions multiple times, but we can use `cache` and do not compute a function with the same input arguments twice.

```
//F(2) = 1
//F(3) = F(2) + F(1)
//F(4) = F(3) + F(2)
//We will compute F(2) twice.
	
constexpr long long fibonaci_noconst(int n){
		if(n <= 1){
				return 1;
		}

		return fibonaci_noconst(n - 1) + fibonaci_noconst(n - 2);
}
```

But if we will use caching this problem will be gone, we store cache in a `std::map<int, long  long>`, where  key is function argument and value is function return value (actually we need to store only 2 previous function invocation results but in current case we store all results).

```
#include <map>
long long _fibonaci_rec(int n, std::map<int, long long>& cache){
    auto cached_value = cache.find(n);
    if(cached_value != cache.end()){
        return cached_value->second;//Return cached value
    }

    auto v1 = _fibonaci_rec(n - 1, cache);
    auto v2 = _fibonaci_rec(n - 2, cache);

    cache[n] = v1 + v2; //store in cache
    return v1 + v2;
}

long long fibonaci_rec(int n){
    std::map<int, long long> cache = {
            {0 , 1}, //F(0) = 1
            {1 , 1}  //F(1) = 1
    };
    return _fibonaci_rec(n, cache);
}
int main(){
    cout<<fibonaci_rec(5); // 8
}
```

**Note** current realization can be easily converted into 1 for-loop, but in the current case, we use this approach, which simplifies understanding of memoization.

**`constexpr`  fibonacci number calculation**

```
constexpr long long fibonaci(int n){
    if(n <= 1){
        return 1;
    }

    return fibonaci(n - 1) + fibonaci(n - 2);
}
...
constexpr auto res = fibonaci(5); //res = 8
```
This code will be compiled with no problem, and the result of evaluation will be evaluated with zero time. But what about calculation Fibonacci number, where `n` is a big number like 40? Actually 40 is not a big number, but result of Fibonaci function with n = 40 will be 165580141.

```
constexpr auto res = fibonaci(40); //Do not compiled
```

I do not really know, maybe this code will be compiled by your compiler, but I have reasonable problems why this code does not compile, let's look what compiler says:

```
1  error: constexpr variable 'res1' must be initialized by a constant expression
      constexpr ll res1 = fibonaci(40);
                   ^      ~~~~~~~~~~~~
2  note: constexpr evaluation hit maximum step limit; possible infinite loop?
    return fibonaci(n - 1) + fibonaci(n - 2);
3  note: in call to 'fibonaci(3)'
4  note: in call to 'fibonaci(4)'
5  note: in call to 'fibonaci(5)'
...
6  note: in call to 'fibonaci(40)'
```

At second line our compiler, says that we hit maximum step limit, it seems our compiler do not cache result of evaluation previous function :( 
Actually, we can compute the complexity of Fibonacci number without caching:

```
T(n) - complexity for calculation Fibonacci for num = n
T(1) = 1
T(0) = 1
T(n) = T(n - 1) + T(n - 2) + O(1)  ; O(1) is sum operation T(n-1) + T(n-2)

Θ(n) = F(n) ; Tight Bound
```

Calculating Fibonacci number without any modification has O(2 ^ n) complexity, cause we have a tree with depth = n, and a total number of nodes is 2 ^ n, **its upper bound**.

But what about our compilation issues? We need compile with flag `-fconstexpr-steps=NUM`, where a num is a number of maximum steps limit. I use **clang-802.0.41**, now we can test our Fibonacci complexity. Let's compute Fibonacci numb for example for num = 10, and set flags  `-fconstexpr-steps` between 2^n and some number lower than 2^n. If program compiled with 2^n and do not compile with a number lower than 2^n, our complexity will be valid.

```
-fconstexpr-steps=1024, n = 10 =>  success compilation
-fconstexpr-steps=600, n = 10 =>  fail compilation

-fconstexpr-steps=512, n = 9 =>  success compilation
-fconstexpr-steps=256, n = 9 =>  fail compilation

-fconstexpr-steps=512, n = 9 =>  success compilation
-fconstexpr-steps=256, n = 9 =>  fail compilation

-fconstexpr-steps=32768, n = 15 =>  success compilation
-fconstexpr-steps=6500, n = 15 =>  fail compilation
```

Yes, my clang compiler does not memoize `constexpr` function invocation! 
But `constexpr` is not only one way to calculate values in precompiled time, we can use c++ template way.

```
template <long long N>
struct Fibonaci{
    enum{value = (Fibonaci<N-1>::value + Fibonaci<N-2>::value)};
};
template <>
struct Fibonaci<0>{
    enum {value = 1};
};
template <>
struct Fibonaci<1>{
    enum {value = 1};
};
template <>
struct Fibonaci<2>{
    enum {value = 1};
};
```

This way will be compiled more faster, but by default recursive template instantiation maximum depth = 1024, if you want increase maximum depth add flag `-ftemplate-depth=N`. In our case it's not nessasery, cause calculation Fibonacci number for N>=1024 is really hard task.


**Calculation time** 

```
#include <iostream>
#include <cmath>
#include <ctime>
#include <vector>
#include <chrono>
using namespace std;
using ll = long long;

constexpr long long fibonaci(ll n){
    if(n <= 1){
        return 1;
    }

    return fibonaci(n - 1) + fibonaci(n - 2);
}
constexpr ll fibonaci_noconst(ll n){
    if(n <= 1){
        return 1;
    }

    return fibonaci_noconst(n - 1) + fibonaci_noconst(n - 2);
}


template <ll N>
struct Fibonaci{
    enum{value = (Fibonaci<N-1>::value + Fibonaci<N-2>::value)};
};
template <>
struct Fibonaci<0>{
    enum {value = 1};
};
template <>
struct Fibonaci<1>{
    enum {value = 1};
};
template <>
struct Fibonaci<2>{
    enum {value = 1};
};


using namespace std;
#include <map>
long long _fibonaci_rec(int n, std::map<int, long long>& cache){
    auto cached_value = cache.find(n);
    if(cached_value != cache.end()){
        return cached_value->second;//Return cached value
    }

    auto v1 = _fibonaci_rec(n - 1, cache);
    auto v2 = _fibonaci_rec(n - 2, cache);

    cache[n] = v1 + v2; //store in cache
    return v1 + v2;
}

long long fibonaci_rec(int n){
    std::map<int, long long> cache = {
            {0 , 1}, //F(0) = 1
            {1 , 1}  //F(1) = 1
    };
    return _fibonaci_rec(n, cache);
}

auto start = std::chrono::system_clock::now();
void startMeasurment(){
    start = std::chrono::system_clock::now();
}
void endMeasurmentTimeAndPrint(std::string method, int n){
    auto end = std::chrono::system_clock::now();
    auto elapsed =
            std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout<< method<<" for num = "<<n<<", total computation time: "<< elapsed.count() <<"ms"<<endl;
}
int main(){
    constexpr ll num = 40;

    startMeasurment();
    constexpr auto f = fibonaci(num);
    endMeasurmentTimeAndPrint("Constexpr Fibonacci",num);

    startMeasurment();
    auto f1 = fibonaci_noconst(num);
    endMeasurmentTimeAndPrint("General Fibonacci",num);

    startMeasurment();
    auto f2 = Fibonaci<num>::value;
    endMeasurmentTimeAndPrint("Template Fibonacci",num);

    startMeasurment();
    auto f3 = fibonaci_rec(num);
    endMeasurmentTimeAndPrint("General Fibonacci with map cache",num);
}
//Compile with flag -fconstexpr-steps=1099162776
```
*Measure time computation for defferent type of calculation fibonaci num*

On my MacBookPro 15 with 2.5 GHz Intel Core i7, it tooks 15minute for **COMPILATION**. You can see results of evaluation bellow.

```
Constexpr Fibonacci for num = 40, total computation time: 0ms
General Fibonacci for num = 40, total computation time: 1580ms
Template Fibonacci for num = 40, total computation time: 0ms
General Fibonacci with map cache for num = 40, total computation time: 0ms

```
















