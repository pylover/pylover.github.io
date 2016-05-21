<!-- 
.. title: Go & Python performance test
.. slug: go-python-performance-test
.. date: 2016-05-22 01:43:09 UTC+04:30
.. tags: benchmark go golang python cython
.. category: programming
.. link: 
.. description: A simple arithmetic test in CPython, Cython & golang
.. type: text
-->

Consider this pseudo code:


    async def worker(worker_idx, iterations):
        result, o = 0, iterations * worker_idx
        for i in range(o):
            result += i**i
        for i in range(o):
            result -= i**i

    await wait([worker(i, 600) for i in range(10)])



