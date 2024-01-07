+++
title = 'Hibernate: First Level Cache'
date = 2024-01-07T09:50:56+02:00
draft = false
+++

**Cache** - area of memory where we store objects that are often accessed saving on unnecessary trips to the database.

There are two levels of caching in Hibernate - **First Level** and **Second Level** both are ID based the
latter being optional.

For simplicity, I will be using the same project structure from [N + 1 Select Problem](/content/posts/n+1/index.md).

### First Level Cache

It's turned on by default and built into the `EntityManager`. Once an object is loaded like `em.find(Student.class,
1);`, a `SELECT` statement will be generated and the fetched entity will be cached for the lifespan of the
**EM**. Follow-up attempts to load the same entity will not result in new `SELECT` statements.

```
final Student johanna = em.find(Student.class, 1);
final Student johanna2 = em.find(Student.class, 1);
System.out.println(johanna == johanna2); // true
```

Hibernate will generate only **once**:

```
Hibernate: 
    select
        s1_0.id,
        s1_0.name,
        t1_0.id,
        t1_0.department,
        t1_0.name 
    from
        students s1_0 
    left join
        teachers t1_0 
            on t1_0.id=s1_0.teacher_id 
    where
        s1_0.id=?
```

If we switch over to using the `SELECT` statement instead we will notice that two `SELECT` statements are
generated. This is because Hibernate does not execute the queries himself but only generates and passes them on to
the database.

```
final Student johanna = em.createQuery("SELECT s from Student s WHERE s.id = :id", Student.class)
                    .setParameter("id", 1)
                    .getSingleResult();
                    
final Student johanna2 = em.createQuery("SELECT s from Student s WHERE s.id = :id", Student.class)
                        .setParameter("id", 1)
                        .getSingleResult();
                        
System.out.println(johanna == johanna2); // true
```

```
Hibernate: 
    select
        s1_0.id,
        s1_0.name,
        s1_0.teacher_id 
    from
        students s1_0 
    where
        s1_0.id=?

Hibernate: 
    select
        s1_0.id,
        s1_0.name,
        s1_0.teacher_id 
    from
        students s1_0 
    where
        s1_0.id=?
```

Despite doing two trips to the database the client has two references pointing to the same object. This is because
Hibernate will notice the instance already exists in the cache (via ID check) and will "drop" the second instance.
The same principles apply when loading collections.  
On a side note you can call `em.refresh(entity)` to force a new `SELECT` statement that will update the cache. 
However this can depend on the *DB Isolation level* which is different for each vendor.

### Benefits

#### Isolation
This provides a good level of isolation since when we load and cache an entity during a 
transaction, even if we select it again shortly after another transaction changes the record in the database we will 
not experience any side effect or data corruption since we will be working with the cached entity during the lifespan of the transaction.

#### Less database trips
Not huge but worth mentioning benefit are the fewer DB trips when Hibernate is using cached instances.

#### Consistency of data

Probably the biggest benefit that no extra
objects are being created in memory due to the ID check. Which means there is no way to corrupt the data by updating 
the wrong instance for an example.