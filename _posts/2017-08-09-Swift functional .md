---
layout:     post
title:      "Swift函数式开发（一）"
date:       2017-08-09 18:04:00
author:     "dingjingpisces"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
    - iOS开发
---


# Swift函数式开发（一）




        let arr = [3,5,6,1,19,5,9,0,-9,2,1,5]
        
        func qsort<T:Comparable>(arr : [T]) -> [T]
        {
            guard let first = arr.first else {
                return []
            }
            return qsort(arr: arr[1...].filter({$0 <= first })) + [first] as [T] + qsort(arr: arr[1...].filter({ $0 > first }))
        }
        print(qsort(arr: arr))

