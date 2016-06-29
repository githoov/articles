A few databases support push notifications and/or triggers. It'd be super cool if Looker could display, say, a single value or time-series visualization and have it update in realtime, without having to refresh. Here's a walkthrough of this process using a local PostgreSQL instance an some node.js:

Step 1. Create a view in the database that aggregates the number of users in our `users` table:
``` sql
create view user_count 
as 
select count(*) as number_of_users 
from users;
```

Step 2. Create a UDF that flips a switch to "on" when a new row has been inserted:
```sql
create or replace function before_each_row_on_users()
returns trigger language plpgsql
as $$
begin
    if TG_OP = 'INSERT' then
      set flags.users to 'on';
    end if;
    return new;
end $$;
```

Step 3. Create a UDF that sends a push notification when our switch is "on" calling from our `user_count` view; and, after the push, it flips the switch back to "off":
```sql
create or replace function after_each_statement_on_users()
returns trigger language plpgsql
as $$
begin
    if (select current_setting('flags.users')) = 'on' then
      perform pg_notify('table_update', json_build_object('number_of_users', number_of_users)::text) as notify 
      from (select number_of_users from user_count) as number_of_users;
      set flags.users to 'off';
    end if;
    return null;
exception
    when undefined_object then
        return null;
end $$;
```

Step 4. Create a trigger that executes the UDF created in (2) upon an insert into `users`:
```sql
create trigger before_each_row_on_users
after insert on users
for each row execute procedure before_each_row_on_users();
```

Step 5. Create a trigger that executes the UDF created in (3) upon an insert into `users`:
```sql
create trigger after_each_statement_on_users
after insert on users
for each statement execute procedure after_each_statement_on_users();
```

Step 6. Fire up a little node server:

```javascript
var pg = require ('pg');

pg.connect("postgres://localhost/test", function(err, client) {
  if(err) {
      console.log(err);
  }
  client.on('notification', function(msg) {
      if (msg.name === 'notification' && msg.channel === 'table_update') {
          var pl = JSON.parse(msg.payload);
          console.log(pl)
      }
  });
  client.query("LISTEN table_update");
});
```

Top window: PostgreSQL `insert`s
Bottom window: Push JSON of `user_count` results
![realtime_dashboard](https://cloud.githubusercontent.com/assets/2467394/14067194/da5c567e-f413-11e5-8a0b-94cd67c47ecc.gif)
