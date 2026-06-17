```dbml
Table users {
  id int [pk]
  name varchar
}

Table orders {
  id int [pk]
  user_id int
}

users.id <> orders.user_id
```



