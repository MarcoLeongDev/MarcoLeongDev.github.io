---
layout: post
title:  "How to create 2d fixed-size array in Swift"
author: marco
categories: [ swift, array, 2d-array, fixed-size, iOS, programming, coding, macOS, tvOS, AppleWatch, two-dimension array ]
image: assets/images/swift banner.png
featured: true
---

Recently I am doing some code tests and found that fixed-size array creation is not as simple in Swift compare to other languages, i.e.

```
// In C++

int array1[64];    // 1-dimension array size 64

int array2[64][64] // 2-dimension array size 64x64

```

After a bit digging, the way to do it in swift:
```
// in Swift:

var array1 = [Int?](repeating: nil, count: 64) // 1 dimension array

var array2 = [[Int?]](
 repeating: [Int?](repeating: nil, count: 64)
 count: 64
) // 2-dimension array size 64x64

// Access it just like normal

array2[4][2] = 42

print(array2[4][2]) // output: 42

```

So a 3d array would be:
```

// in C++

int array3[3][3][3];

// in Swift !!!!!!

var array3 = 
[[[Int?]]](
  repeating: [[Int?]](
    repeating: [Int?](
      repeating: var,
      count: 3),
  count: 3),
count: 3)

```

Now you know how, you can try 7-dimension, happy coding 🍳


