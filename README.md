A paginator doing cursor-based pagination based on [GORM](https://github.com/go-gorm/gorm) and [GORM-CURSOR-PAGINATION](https://github.com/pilagod/gorm-cursor-paginator)
Supports cursor navigation using AFTER/BEFORE cursor setup and FIRST/LAST number of rows

Given an `User` model for example:

```go
type User struct {
    ID          int
    JoinedAt    time.Time `gorm:"column:created_at"`
}
```

We first need to create a `paginator.Paginator` for `User`, here are some useful patterns:

1. Configure by `paginator.Option`, those functions with `With` prefix are factories for `paginator.Option`:

    ```go
    func CreateUserPaginator(
        cursor paginator.Cursor,
        order *paginator.Order,
        first *int,
        last *int,
    ) *paginator.Paginator {
        opts := []paginator.Option{
            &paginator.Config{
                Keys: []string{"ID", "JoinedAt"},
                First: 10,
                Order: paginator.ASC,
            },
        }
        if first != nil {
            opts = append(opts, paginator.WithFirst(*first))
        }
        if last != nil {
            opts = append(opts, paginator.WithLast(*last))
        }
        if order != nil {
            opts = append(opts, paginator.WithOrder(*order))
        }
        if cursor.After != nil {
            opts = append(opts, paginator.WithAfter(*cursor.After))
        }
        if cursor.Before != nil {
            opts = append(opts, paginator.WithBefore(*cursor.Before))
        }
        return paginator.New(opts...)
    }
    ```

2. Configure by setters on `paginator.Paginator`:

    ```go
    func CreateUserPaginator(
        cursor paginator.Cursor,
        order *paginator.Order,
        first *int,
        last *int,
    ) *paginator.Paginator {
        p := paginator.New(
            &paginator.Config{
                Keys: []string{"ID", "JoinedAt"},
                First: 10,
                Order: paginator.ASC,
            },
        )
        if order != nil {
            p.SetOrder(*order)
        }
        if first != nil {
            p.SetFirst(*first)
        }
        if last != nil {
            p.SetLast(*last)
        }
        if cursor.After != nil {
            p.SetAfter(*cursor.After)
        }
        if cursor.Before != nil {
            p.SetBefore(*cursor.Before)
        }
        return p
    }
    ```

3. Configure by `paginator.Rule` for fine grained setting for each key:

    > Please refer to [Specification](#specification) for details of `paginator.Rule`.

    ```go
    func CreateUserPaginator(/* ... */) {
        p := paginator.New(
            &paginator.Config{
                Rules: []paginator.Rule{
                    {
                        Key: "ID",
                    },
                    {
                        Key: "JoinedAt",
                        Order: paginator.DESC,
                        SQLRepr: "users.created_at",
                        NULLReplacement: "1970-01-01",
                    },
                },
                First: 10,
                // Order here will apply to keys without order specified.
                // In this example paginator will order by "ID" ASC, "JoinedAt" DESC.
                Order: paginator.ASC, 
            },
        )
        // ...
        return p
    }
    ```

After knowing how to setup the paginator, we can start paginating `User` with GORM:

```go
func FindUsers(db *gorm.DB, query Query) ([]User, paginator.Cursor, error) {
    var users []User

    // extend query before paginating
    stmt := db.
        Select(/* fields */).
        Joins(/* joins */).
        Where(/* queries */)

    // create paginator for User model
    p := CreateUserPaginator(/* config */)

    // find users with pagination
    result, cursor, err := p.Paginate(stmt, &users)

    // this is paginator error, e.g., invalid cursor
    if err != nil {
        return nil, paginator.Cursor{}, err
    }

    // this is gorm error
    if result.Error != nil {
        return nil, paginator.Cursor{}, result.Error
    }

    return users, cursor, nil
}
```

The second value returned from `paginator.Paginator.Paginate` is a `paginator.Cursor` struct, which is same as `cursor.Cursor` struct:

```go
type Cursor struct {
    After  *string `json:"after" query:"after"`
    Before *string `json:"before" query:"before"`
}
```

That's all! Enjoy paginating in the GORM world. :tada:

> For more paginating examples, please checkout [exmaple/main.go](https://github.com/pilagod/gorm-cursor-paginator/blob/master/example/main.go) and [paginator/paginator_paginate_test.go](https://github.com/pilagod/gorm-cursor-paginator/blob/master/paginator/paginator_paginate_test.go)
>
> For manually encoding/decoding cursor exmaples, please checkout [cursor/encoding_test.go](https://github.com/pilagod/gorm-cursor-paginator/blob/master/cursor/encoding_test.go)

## Specification

### paginator.Paginator

Default options used by paginator when not specified:

- `Keys`: `[]string{"ID"}`
- `First`: `10`
- `Order`: `paginator.DESC`

### paginator.Rule

- `Key`: Field name in target model struct.
- `Order`: Order for this key only.
- `SQLRepr`: SQL representation used in raw SQL query.<br/>
    > This is especially useful when you have `JOIN` or table alias in your SQL query. If `SQLRepr` is not specified, paginator will get table name from model, plus table key derived by below rules to form the SQL query:
    > 1. Find GORM tag `column` on struct field.
    > 2. If tag not found, convert struct field name to snake case.
- `NULLReplacement`: Replacement for NULL value when paginating by nullable column.<br/>
    > If you paginate by nullable column, you will encounter [NULLS { FIRST | LAST } problems](https://learnsql.com/blog/how-to-order-rows-with-nulls/). This option let you decide how to order rows with NULL value. For instance, we can set this value to `1970-01-01` for a nullable `date` column, to ensure rows with NULL date will be placed at head when order is ASC, or at tail when order is DESC.

## Changelog

### v1.0.0

- Add `first/last` pagination instead of `limit` 
- 
## License

© Dmitry Krasnikov (mainpart), 2022-NOW

Released under the [MIT License](https://github.com/pilagod/gorm-cursor-paginator/blob/master/LICENSE)
