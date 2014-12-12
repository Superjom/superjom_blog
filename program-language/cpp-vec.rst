用C++实现向量运算
====================
之前论文实验有需要实现一个语言模型，中间需要用到向量运算。

考虑到功能需求比较简单，就自己写了要一个类，这里做一个总结。 

迭代器
------
首先是迭代器的实现

.. code-block:: c
    :linenos:
    
    template<class T>
    class VecIterator {
        T *ptr;
        // 构造函数
        VecIterator(const VecIterator &other) : ptr(other.ptr) {}
        VecIterator(T p=0) : ptr(p) {}

        // 元素存取
        T& operator*() const { return *ptr; }
        T* operator->() const { return ptr; }

        // 操作符重载
        VecIterator& operator++() {
            ptr++; return *this;
        }

        VecIterator& operator=(const VecIterator &other) {
            ptr = other.ptr;
            return *this;
        }

        VecIterator operator- (int i) {
            return VecIterator(ptr-i);
        }

        bool operator==(const VecIterator &other) const {
            return ptr == other.ptr;
        }

        bool operator!=(const VecIterator &other) const {
            return ptr != other.ptr;
        }

    }; // end class VecIterator


Vec类实现
-----------
基本类型定义
**************

.. code-block:: c
    :linenos:
    
    template <class T, int SIZE>
    class Vec {
    public:
        // 类型定义
        typedef T value_type;
        typedef unsigned int index_type;
        typedef VecIterator iterator;

这里考虑到不同长度的向量应该认为是两种类型，因此将向量的长度用模板进行指定。

如此，可以类似如下的类型 ：

.. code-block:: c
    :linenos:

    typedef Vec<int, 7> vec7_t;
    

私有成员
********
由于长度已经用模板制定，这里只有一个数据的指针作为私有成员。

.. code-block:: c
    :linenos:

    private:
        T *vec;
    }; // end class Vec

构造函数    
**************
.. code-block:: c
    :linenos:
    
        // 混合的默认构造函数
        Vec(T *arr=NULL) : vec(arr) {
            if(!arr) {
                vec = new T[SIZE];
            }
        }

        // 拷贝构造函数
        Vec(const Vec &other) : vec(NULL) {
            assert(other.size() == SIZE);
            if(vec) delete vec;

            vec = new T[SIZE];
            memcpy(vec, other.getVec(), SIZE*sizeof(T));
        }

析构函数
***********
.. code-block:: c
    :linenos:

        ~Vec() {
            if(vec) delete vec;
        }


功能性函数
***********
.. code-block:: c
    :linenos:

        T sum() const {
            T total = 0;
            for(index_type i=0; i<SIZE; i++) {
                total += vec[i];
            }
            return total;
        }

        float mean() const {
            return  float(sum()) / SIZE;
        }

        // 内积
        value_type dot(Vec &other) const {
            assert(other.size() == SIZE); 
            T total = 0;
            for (index_type i=0; i<SIZE; i++) {
                total += vec[i] * other[i];
            }
            return total;
        }

        // 外积
        Vec operator* (const Vec &other) const {
            assert(other.size() == SIZE);
            Vec<T, SIZE> newVec;
            for (index_type i=0; i<SIZE; i++) {
                newVec[i] = vec[i] * other[i];
            }
            return newVec;
        }

        // L2 归一化 ， 只对浮点数有效
        Vec& norm() {
            T total = 0;
            for (index_type i=0; i<SIZE; i++) {
                total += vec[i] * vec[i];
            }
            total = sqrt(total);
            for (index_type i=0; i<SIZE; i++) {
                vec[i] /= total;
            }
            return *this;
        }

其他操作符重载
-----------------
大部分操作符都会为向量类型和单个元素做重载

.. code-block:: c
    :linenos:
    
        T& operator[] (index_type id) {
            return vec[id];
        }

        const T& operator[] (index_type id) const {
            return vec[id];
        }

        Vec& operator= (const Vec& other) {
            assert(other.size() == SIZE);
            memcpy(vec, other.size(), SIZE*sizeof(T));
            return *this;
        }

        Vec& operator= (T val) {
            for(index_type i=0; i<SIZE; i++) {
                vec[i] = val;
            }
            return *this;
        }
        
        Vec& operator+= (const Vec& other) {
            assert(other.size() == SIZE);
            for(index_type i=0; i<SIZE; i++) {
                vec[i] += other[i];
            }
            return *this;
        }

        Vec& operator+= (T val) {
            for(index_type i=0; i<SIZE; i++) {
                vec[i] += val;
            }
            return *this;
        }

        Vec& operator-= (const Vec& other) {
            assert(other.size() == SIZE);
            for(index_type i=0; i<SIZE; i++) {
                vec[i] -= other[i];
            }
            return *this;
        }

        Vec& operator-= (T val) {
            for(index_type i=0; i<SIZE; i++) {
                vec[i] -= val;
            }
            return *this;
        }

        Vec<T,SIZE>& operator*= (T val) {
            for(index_type i=0; i<SIZE; i++) {
                vec[i] *= val;
            }
            return *this;
        }

        Vec<T,SIZE>& operator*= (T val) {
            for(index_type i=0; i<SIZE; i++) {
                vec[i] *= val;
            }
            return *this;
        }

        // 外积
        Vec<T,SIZE>& operator*= (const Vec &other) {
            for(index_type i=0; i<SIZE; i++) {
                vec[i] *= other[i];
            }
            return *this;
        }

        Vec& operator/= (float val) {
            for(index_type i=0; i<SIZE; i++) {
                vec[i] /= val;
            }
            return *this;
        }

        friend Vec<T, SIZE> operator+ (const Vec &vec, T val) {
            Vec<T, SIZE> newVec(vec);
            newVec += val;
            return newVec;
        }

        friend Vec(T, SIZE> operator* (const Vec &vec, T val) {
            Vec<T, SIZE> newVec(vec);
            newVec *= val;
            return newVec;
        }

迭代器首尾
------------
.. code-block:: c
    :linenos:

    iterator begin() {
        return vec;
    }

    iterator end() {
        return vec + SIZE;
    }


