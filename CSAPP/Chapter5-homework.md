# 优化程序性能 家庭作业
### 5.13
A. 图略，不过需要注意一些要点：
首先是vmulsd和vaddsd是两个分开处理的，换句话说，可以并行。vmovsd和vmulsd是串行处理的，之后是rcx和rbx，这是第三条线。
B. 3
C. 1
D. 乘法不在关键路径上，因此乘法可以流水线工作。
### 5.14
```C++
void inner4(vec_ptr u, vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(u);
    long limit = length - 5;
    data_t *udata = get_vec_start(u);
    data_t *vdata = get_vec_start(v);
    data_t sum = (data_t)0;
    for (i = 0; i < limit; i += 6)
    {
        sum = sum + udata[i] * vdata[i];
        sum = sum + udata[i + 1] * vdata[i + 1];
        sum = sum + udata[i + 2] * vdata[i + 2];
        sum = sum + udata[i + 3] * vdata[i + 3];
        sum = sum + udata[i + 4] * vdata[i + 4];
        sum = sum + udata[i + 5] * vdata[i + 5];
    }
    for (; i < length; i++)
        sum = sum + udata[i] * vdata[i];
    *dest = sum;
}
```
### 5.15
```C++
void inner4(vec_ptr u, vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(u);
    long limit = length - 5;
    data_t *udata = get_vec_start(u);
    data_t *vdata = get_vec_start(v);
    data_t sum1 = (data_t)0;
    data_t sum2 = (data_t)0;
    data_t sum3 = (data_t)0;
    data_t sum4 = (data_t)0;
    data_t sum5 = (data_t)0;
    data_t sum6 = (data_t)0;
    for (i = 0; i < limit; i += 6)
    {
        sum1 = sum1 + udata[i] * vdata[i];
        sum2 = sum2 + udata[i + 1] * vdata[i + 1];
        sum3 = sum3 + udata[i + 2] * vdata[i + 2];
        sum4 = sum4 + udata[i + 3] * vdata[i + 3];
        sum5 = sum5 + udata[i + 4] * vdata[i + 4];
        sum6 = sum6 + udata[i + 5] * vdata[i + 5];
    }
    for (; i < length; i++)
        sum1 = sum1 + udata[i] * vdata[i];
    *dest = sum1 * sum2 * sum3 * sum4 * sum5 * sum6;
}
```
### 5.16
```C++
void inner4(vec_ptr u, vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(u);
    long limit = length - 5;
    data_t *udata = get_vec_start(u);
    data_t *vdata = get_vec_start(v);
    data_t sum = (data_t)0;
    for (i = 0; i < limit; i += 6)
    {
        sum = sum + (udata[i] * vdata[i] + udata[i + 1] * vdata[i + 1] + udata[i + 2] * vdata[i + 2] + udata[i + 3] * vdata[i + 3] + udata[i + 4] * vdata[i + 4] + udata[i + 5] * vdata[i + 5]);
    }
    for (; i < length; i++)
        sum = sum + udata[i] * vdata[i];
    *dest = sum;
}
```
### 5.17
```C++
void *basic_memset(void *s, int c, size_t n)
{
    size_t cnt = 0, k = sizeof(unsigned long);
    unsigned long cl;
    unsigned char *schar = s, *cchar = (unsigned char *)&cl;
    for (int i = 0; i < k; i++)
        cchar[i] = (unsigned char)c;
    if (n < k)
        while (cnt++ < n)
            *schar++ = (unsigned char)c;
    else
    {
        while ((size_t)schar % k)
        {
            *schar++ = (unsigned char)c;
            cnt++;
        }
        while (cnt < n - k)
        {
            for (int j = 0; j < (k >> 2); j++)
            {
                schar[0] = cchar[0];
                schar[1] = cchar[1];
                schar[2] = cchar[2];
                schar[3] = cchar[3];
                schar += 4;
                cnt += 4;
            }
        }
        while (cnt++ < n)
            *schar++ = (unsigned char)c;
    }
    return s;
}
```
### 5.18
第一个是自己写的，第二个是[@yang_f_k](https://blog.csdn.net/yang_f_k/article/details/9471639)的，本来一看第二个写错了，实际上，利用交错的乘法来完成这个事情恰到好处。
```C++
double polyh2(double a[], double x, int degree)
{
    long int i = 0;
    double result1 = 0, result2 = 0, result3 = 0;
    double x2 = x * x, x3 = x * x * x;
    double xwpr = 1;
    for (; i <= degree - 8; i += 9)
    {
        result1 = result1 + (a[i] + a[i + 1] * x + a[i + 2] * x2) * xwpr;
        xwpr *= x3;
        result2 = result2 + (a[i + 3] + a[i + 4] * x + a[i + 5] * x2) * xwpr;
        xwpr *= x3;
        result3 = result3 + (a[i + 6] + a[i + 7] * x + a[i + 8] * x2) * xwpr;
        xwpr *= x3;
    }
    double result = result1 + result2 + result3;
    for (; i <= degree; i++)
    {
        result = result + a[i] * xwpr;
        xwpr *= x;
    }
    return result;
}
```
```C++
double polyh3(double a[], double x, int degree)
{
    long int i;
    double x2 = x * x;
    double x3 = x2 * x;
    double result1, result2, result3 = a[degree];
    for (i = degree - 1; i >= 9; i -= 9)
    {
        result1 = result3 * x3 + (a[i - 2] + x * a[i - 1] + x2 * a[i]);
        result2 = result1 * x3 + (a[i - 5] + x * a[i - 4] + x2 * a[i - 3]);
        result3 = result2 * x3 + (a[i - 8] + x * a[i - 7] + x2 * a[i - 6]);
    }
    for (; i >= 0; i--)
        result3 = a[i] + x * result3;
    return result3;
}
```
### 5.19
同样[@yang_f_k](https://blog.csdn.net/yang_f_k/article/details/9471639)，类似的有一个和5.18类似的版本，但是仔细想想似乎写出来没什么意义，性能也不如下面的这个版本。
不说了我去买4980HQ了#滑稽。
```C++
void psum3(float a[], float p[], long int n)
{
    long int i;
    p[0] = a[0];
    float val1, val2, val3 = p[0];
    for (i = 1; i < n - 2; i += 3)
    {
        val1 = val3 + a[i];
        val2 = val1 + a[i + 1];
        val3 = val2 + a[i + 2];
        p[i] = val1;
        p[i + 1] = val2;
        p[i + 2] = val3;
    }
    for (; i < n; i++)
        p[i] = p[i - 1] + a[i];
}
```