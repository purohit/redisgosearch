# redisgosearch

redisgosearch implements fast full-text search with Golang and Redis, using Redis's rich support for sets.

This is a fork of the original package by The Plant and sunfmin. This fork has Chinese character support and cleaned-up interfaces with documentation. The tests also have no external dependencies other than a working Redis installation.

The [original documentation](https://theplant.jp/en/blogs/13-techforce-making-a-simple-full-text-search-with-golang-and-redis) is clarified below.

## Usage

Let's say you have blog entries:

```go
type Entry struct {
    Id          string
    Title       string
    Content     string
}
```

You want a user to be able to search any word or combination of `Title` or `Content`. Two data samples are:

```go
Entry {
    Id:      "50344415ff3a8aa694000001",
    Title:   "Organizing Go code",
    Content: "Go code is organized differently from that of other languages. This post discusses",
}

Entry {
    Id:      "50344415ff3a8aa694000002",
    Title:   "Getting to know the Go community",
    Content: "Over the past couple of years Go has attracted a lot of users and contributors",
}
```

In order to let people search any word in these two entries, we first index these texts into the Redis database as keywords that we segmented from the title and content as keys, with the Ids as Redis set values.

```
redis 127.0.0.1:6379> keys *
1) "entries:keywords:go"
2) "entries:keywords:community"
3) ...

redis 127.0.0.1:6379> SMEMBERS entries:keywords:go
1) "entries:entity:50344415ff3a8aa694000001"
2) "entries:entity:50344415ff3a8aa694000002"

redis 127.0.0.1:6379> SMEMBERS entries:keywords:community
1) "entries:entity:50344415ff3a8aa694000002"
```

Then, for example, if the user types the keywords go community, we first segment the keywords to `["go", "community"]` then do:

```
redis 127.0.0.1:6379> SINTER entries:keywords:go entries:keywords:community
1) "entries:entity:50344415ff3a8aa694000002"
```

With the Redis Intersect command `SINTER` for Set, we are able to get the entry ids for entries that contain both the keywords go and community.

redisgosearch can index any Go object that satisfies `Indexable`. You can `Search` from the indexed Redis database, and unmarshal results into your indexed objects.

```go
type Entry struct {
    Id          string
    GroupId     string
    Title       string
    Content     string
    Attachments []*Attachment
}

func (entry *Entry) IndexPieces() (r []string, ais []redisgosearch.Indexable) {
    r = append(r, entry.Title)
    r = append(r, entry.Content)

    for _, a := range entry.Attachments {
    r = append(r, a.Filename)
        ais = append(ais, &IndexedAttachment{entry, a})
    }

    return
}

func (entry *Entry) IndexEntity() (indexType string, key string, entity interface{}) {
    key = entry.Id
    indexType = "entries"
    entity = entry
    return
}

func (entry *Entry) IndexFilters() (r map[string]string) {
    r = make(map[string]string)
    r["group"] = entry.GroupId
    return
}
```

`IndexPieces` tells the package what text needs to be segmented and indexed. Note that in an `Entry` struct, you might also want to index other types of data that are connected to the entry, for example attachment data. In our case, the user can search any filename and find out which entries those files belong to. So, the other return values return an array of `Indexable` objects that can be indexed and connected together.

`IndexEntity` tells the package the type of index from the namespace and the key stored in the Redis Set. The actual entity value will be marshalled into JSON and stored into Redis.

`IndexFilters` gives the ability to add additional metadata that can be filtered when searching. For example I want to search “go community” in “New York”.

```go
var entries []*Entry
count, err := client.Search("entries", "go community", map[string]string{"group": "New York"}, 0, 20, &entries)
```

The 0 and 20 is for pagination, and `count` is the total number of entries that matched "go community".

## Contributing
The current feature set is simple, and new features are appreciated. Please initiate a pull request, and make sure to `go fmt` and `golint`!
