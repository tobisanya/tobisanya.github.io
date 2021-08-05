---  
layout: post  
title:  "Data isolation in a pooled multi-tenant environment - PostgreSQL RLS"  
date:   2021-08-05 06:45:00  
categories: [database, multi-tenancy, security, postgresql]  
comments: true  
excerpt: "Easily and securely prevent leakage of data in a pooled multi-tenant setup"

---
![data security image](https://cdn.pixabay.com/photo/2017/03/23/12/56/security-2168233_1280.jpg)

Introduction
------------  

When designing a multi-tenant architecture, one very important piece of the puzzle is deciding how the data of tenants will be isolated.

Before proceeding, let me clarify that when I say data, I mean information stored in a database, specifically an RDBMS.

Our goal is to have every tenant feel like the application is exclusive to them, both in terms of efficiency and security. One mistake we want to avoid at all costs is the accidental exposure of data between tenants. The business may be put at risk for a diverse range of legal and financial problems that could eventually ground the business to a halt. So yeah, this is something we should take seriously.

Multi-tenant setups employ a variety of approaches to the isolation of data. Among the most common types are the Silo, Bridge, and Pool. This post will focus on setting up a secure, and easy-to-manage data isolation pattern in a pooled database model, using the PostgreSQL RDBMS.

What is a Pooled data isolation model?
--------------------------------------------------------------
Without briefly describing the other models, it may be difficult to describe what a pooled data isolation model is.

The **Silo** model is a database-per-tenant model, in which we have separate instances of the database set up for each tenant.

In the **Bridge** model, we use the same database instance for all tenants but we separate them at the schema level. Tenants each have their own dedicated tables

In the **Pool** model, all tenants share a single database and schema. All tables are also shared. This means we need to add a discriminator/partition field to the shared tables in order to control access to a specific tenant's data. The pool model greatly simplifies the multi-tenancy design. Infrastructure can be managed more easily. In addition, updating our application becomes very easy. The maintenance costs are also lower.

If you are interested in taking a deeper dive into the different isolation models, especially how they can be set up in different AWS services, [this whitepaper by the AWS SaaS Factory team](https://d1.awsstatic.com/whitepapers/Multi_Tenant_SaaS_Storage_Strategies.pdf) does a good job of breaking things down further

How do we isolate data in a pooled isolation model?
--------------------------------------------------------------
Let's build a scenario. We are designing a SaaS application where companies can manage their employees, an HRIS system. We have a table, `staff` that will contain the information of company staff. Multiple companies will use our application meaning the `staff`  table will contain information about people from different companies. How do we ensure that HR of *Company A* doesn't see the data of another company?

### The traditional way.
The quickest answer to this question is to add a column, say `company_id`, to our table. We then need to ensure that every query hitting this table always contains the condition `WHERE company_id=[company_id_of_user_making_the_request]`

This works but has many security and maintainability issues. Our trust rests in the application code, and we hope that SQL queries interacting with this table contain that clause and that the clause has been properly set. It is too susceptible to errors, a simple mistake can easily lead to data leaks across tenants, and as we said, this is something we would like to avoid at all costs.

It'll be great if the software developers can issue a simple `SELECT * from staff`
and not have to worry about multi-tenancy. In this way, we greatly reduce the possibility of errors, and our application is easier to maintain. Enter PostgreSQL RLS.

### The RLS way.
From version 9.5, PostgreSQL included a feature called **Row Level Security (RLS)**. By default, RLS is a feature that allows us to define policies on the table to control how data in that table can be accessed. The best way to explain it is to see it in action.

Lets create a table:

    CREATE  TABLE "staff" ( 
    "owner" character varying NOT NULL DEFAULT current_user, 
    "name" character varying NOT NULL
    );

Note the field `owner`. It will act as our discriminator column. It will also be automatically filled with the username of the current DB connection.

Then we enable RLS on the table
``ALTER  TABLE staff ENABLE  ROW  LEVEL SECURITY;``

If we stop at this stage, a default-deny policy will be used by PostgreSQL, meaning that no rows will be visible when we try to query the database.
We have to let PostgreSQL know how we want to restrict access by creating a row security policy. Let's do this next.

    CREATE POLICY data_isolation_policy 
    ON staff  
    USING (owner = current_user);

*Note that `data_isolation_policy` is just a name given to the policy. You can choose whatever name you desire.* *Also note that some roles may not be subject to RLS. From the PostgreSQL docs:*

> Superusers and roles with the BYPASSRLS attribute always bypass the row security system when accessing a table. Table owners normally bypass row security as well, though a table owner can choose to be subject to row security with ``ALTER TABLE ... FORCE ROW LEVEL SECURITY``.

By creating the policy above on our table, we are telling PostgreSQL to only allow access (SELECT, INSERT, UPDATE, DELETE) to rows where the column `owner` matches the username of the current connection. Think of it as the database system automatically adding the `WHERE` clause to every query on that table for us.

This means we have effectively centralized the isolation logic and have assigned it to the database system to handle. Developers can now continue writing their queries as though this is a single-tenant system while the database handles the isolation logic. This is a lot safer and easier to maintain.

### Single DB Connection for all tenants
One thing you may have noticed in the solution above is that it requires us to maintain separate database users (roles) for each of our tenants. This may not scale well, especially if you have to manage hundreds of tenants.

You can still use PostgreSQL RLS without going through the pain of maintaining database users for each of your tenants.

On every connection, a runtime parameter will need to be defined, similar to a connection session variable, which will contain the current tenant context.

The first step is to ensure that every connection to the database contains this context information. Here's an example using [TypeORM](https://github.com/typeorm/typeorm)

    await connection.query(`SET app.current_tenant = '${tenantId}'`);

The next step is to modify the policy code above to

    CREATE POLICY data_isolation_policy 
    ON tenant 
    USING (owner = current_setting('app.current_tenant')::INT);

The change happening here is that instead of comparing the discriminator column to the system-defined `current_user` variable, we compare to the configuration variable that we set

*Note that this method gets more low-level than the approach described above and can get very complicated if you don't have direct management of your connection pooling. This is because the configuration query above needs to be run every time a connection is created or resolved from the pool.*

## Conclusion
As we have discussed, the pool data isolation model makes a multi-tenant application easier and a lot cheaper to maintain because we re-use the same infrastructure for all tenants.

Furthermore, We found that co-locating data of different tenants/companies means we have to be more vigilant since there is an increased risk of data leakage across tenants.

We then discussed how the RLS feature of PostgreSQL helps us simplify the way we securely access and modify our data.

To learn more about PostgresRLS you can check out its [documentation](https://www.postgresql.org/docs/9.5/ddl-rowsecurity.html).
Also, if you are interested in more data isolation patterns, especially how you can implement them on cloud computing platforms, check out [this whitepaper by the AWS SaaS Factory team](https://d1.awsstatic.com/whitepapers/Multi_Tenant_SaaS_Storage_Strategies.pdf)
