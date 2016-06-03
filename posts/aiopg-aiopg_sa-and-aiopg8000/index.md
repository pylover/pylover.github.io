<!-- 
.. title: aiopg, aiopg.sa and aiopg8000
.. slug: aiopg-aiopg_sa-and-aiopg8000
.. date: 2016-06-04 03:05:46 UTC+04:30
.. tags: aiopg,aiopg.sa,aiopg8000,asyncio,python,postgresql
.. category: programming
.. link: 
.. description: benchmarking aiopg, aiopg.sa & aiopg8000.
.. type: text
-->


Why?:
-----
There are two available async adaptors for postgreSQL in python: aiopg, aiopg.sa & aiopg8000

So, benchmarking these packages is the first step before using one of those.

Last result with 90 workers and inserting 50 rows per worker:

        [aiopg.sa]	Total time: 12.46s
        [aiopg]	    Total time: 5.57s
        [aiopg8000]	Total time: 40.83s
        [aiopg]     is 2.24x faster than [aiopg.sa]
        [aiopg]     is 7.33x faster than [aiopg8000]
        [aiopg.sa]  is 3.28x faster than [aiopg8000]


Code
----

To run the code, create databases and install dependencies first:

    $ sudo -u postgres createdb aiopg8000
    $ sudo -u postgres createdb aiopg
    $ sudo -u postgres createdb aiopg_sa
    $ pip install aiopg aiopg8000 sqlalchemy
    

Here is the code:

    import asyncio
    import datetime
    import aiopg
    # noinspection PyPackageRequirements
    import aiopg8000
    import sqlalchemy as sa
    from aiopg.sa import create_engine
    
    ROWS_PER_WORKER = 50
    WORKERS_COUNT = 90
    
    
    #####################################
    # aiopg8000
    #####################################
    
    
    async def aiopg8000_stream():
        return await asyncio.open_connection(
            host='127.0.0.1',
            port=5432,
            # proto=socket.IPPROTO_TCP,
            ssl=False)
    
    
    async def aiopg8000_create_connection():
        return await aiopg8000.connect(
            stream_generator=aiopg8000_stream,
            user='postgres',
            password='postgres',
            database='aiopg8000'
        )
    
    
    async def aiopg8000_setup_db():
        db = await aiopg8000_create_connection()
        c = await db.cursor()
        await c.execute('DROP TABLE IF EXISTS t1')
        await c.execute(
            'CREATE TABLE t1('
            '   id    SERIAL  NOT NULL PRIMARY KEY, '
            '    age int not null, '
            '    name varchar(50) null)'
        )
        await db.commit()
        await c.yield_close()
        await db.yield_close()
    
    
    async def aiopg8000_worker(name: str):
        db = await aiopg8000_create_connection()
        c = await db.cursor()
        for i in range(ROWS_PER_WORKER):
            await c.execute(
                'INSERT INTO t1(age, name) VALUES(%s, %s ) RETURNING id, age, name', (i, name)
            )
            results = await c.fetchall()
            for row in results:
                _id, age, name = row
                if _id % 100 == 0:
                    print('id = %s, age = %s, name = %s' % (_id, age, name))
            await db.commit()
            # time.sleep(.3)
    
        await c.yield_close()
        await db.yield_close()
    
    
    ####################################
    # aiopg
    ####################################
    
    aiopg_dsn = 'dbname=aiopg user=postgres password=postgres host=127.0.0.1'
    aiopg_pool = None
    
    async def aiopg_setup_db():
        global aiopg_pool
        aiopg_pool = await aiopg.create_pool(aiopg_dsn)
        async with aiopg_pool.acquire() as conn:
            async with conn.cursor() as c:
                await c.execute('DROP TABLE IF EXISTS t1')
                await c.execute(
                    'CREATE TABLE t1('
                    '   id    SERIAL  NOT NULL PRIMARY KEY, '
                    '    age int not null, '
                    '    name varchar(50) null)'
                )
    
    
    async def aiopg_worker(name: str):
        async with aiopg_pool.acquire() as conn:
            async with conn.cursor() as c:
                for i in range(ROWS_PER_WORKER):
                    await c.execute(
                        'INSERT INTO t1(age, name) VALUES(%s, %s ) RETURNING id, age, name', (i, name)
                    )
                    results = await c.fetchall()
                    for row in results:
                        _id, age, name = row
                        if _id % 100 == 0:
                            print('id = %s, age = %s, name = %s' % (_id, age, name))
    
    
    ########################################
    # aiopg.sa
    ########################################
    
    aiopg_sa_metadata = sa.MetaData()
    aiopg_sa_t1 = sa.Table(
        't1', aiopg_sa_metadata,
        sa.Column('id', sa.Integer, primary_key=True),
        sa.Column('age', sa.Integer),
        sa.Column('name', sa.String(50))
    )
    aiopg_sa_engine = None
    
    async def aiopg_sa_setup_db():
        global aiopg_sa_engine
        aiopg_sa_engine = await create_engine(
            user='postgres',
            database='aiopg_sa',
            host='127.0.0.1',
            password='postgres')
        async with aiopg_sa_engine.acquire() as conn:
            await conn.execute('DROP TABLE IF EXISTS t1')
            await conn.execute(
                'CREATE TABLE t1('
                '   id    SERIAL  NOT NULL PRIMARY KEY, '
                '    age int not null, '
                '    name varchar(50) null)'
            )
    
    
    async def aiopg_sa_worker(name: str):
        async with aiopg_sa_engine.acquire() as conn:
            for i in range(ROWS_PER_WORKER):
                results = await conn.execute(aiopg_sa_t1.insert().values(age=i, name=name))
                for row in results:
                    _id = row[0]
                    if _id % 100 == 0:
                        print('id = %s, age = %s, name = %s' % (_id, i, name))
            conn.close()
    
    
    async def aiopg_sa_teardown():
        aiopg_sa_engine.close()
        await aiopg_sa_engine.wait_closed()
    
    
    ########################################
    # MAIN
    ########################################
    
    async def get_package_functions(package_name: str):
        setup_db = eval('%s_setup_db' % package_name)
        worker = eval('%s_worker' % package_name)
        try:
            teardown = eval('%s_teardown' % package_name)
        except NameError:
            teardown = None
        return setup_db, worker, teardown
    
    
    async def run_benchmark(package_name: str):
        setup_db, worker, teardown = await get_package_functions(package_name)
    
        try:
            await setup_db()
            start = datetime.datetime.now()
            await asyncio.gather(*[worker('worker: %s' % i) for i in range(WORKERS_COUNT)])
            delta = (datetime.datetime.now() - start).total_seconds()
            if teardown is not None:
                await teardown()
            return delta
        except KeyboardInterrupt:
            print('CTRL+C Pressed, Terminating.')
            return
    
    
    async def main():
        aiopg_sa_delta = await run_benchmark('aiopg_sa')
        aiopg_delta = await run_benchmark('aiopg')
        aiopg8000_delta = await run_benchmark('aiopg8000')
        print('[aiopg.sa]\tTotal time: %.2Fs' % aiopg_sa_delta)
        print('[aiopg]\tTotal time: %.2Fs' % aiopg_delta)
        print('[aiopg8000]\tTotal time: %.2Fs' % aiopg8000_delta)
        print('[aiopg] is %.2Fx faster than [aiopg.sa]' % (aiopg_sa_delta / aiopg_delta))
        print('[aiopg] is %.2Fx faster than [aiopg8000]' % (aiopg8000_delta / aiopg_delta))
        print('[aiopg.sa] is %.2Fx faster than [aiopg8000]' % (aiopg8000_delta / aiopg_sa_delta))
        print('Bye Bye !')
    
    
    if __name__ == '__main__':
        loop = asyncio.get_event_loop()
        loop.run_until_complete(main())

