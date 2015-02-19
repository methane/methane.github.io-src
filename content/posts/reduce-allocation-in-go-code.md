---
title: "Reduce allocation in Go code"
date: "2015-02-18"
categories:
    - "go"
---

I've implemented parameter interpolation in [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql). You can enable it by adding "interpolateParams=true" option to dsn.

Why this feature is important is described in [here](https://vividcortex.com/blog/2014/11/19/analyzing-prepared-statement-performance-with-vividcortex/) and [here](https://eng.uservoice.com/blog/2015/01/28/introducing-gocraft/dbr/). I don't say lot about it here.

When writing low level library like mysql driver, you can't assume user's performance requiremnt. Especially, avoiding memory allocations is important. Go's GC is not so fast. And everyone want memory usage of their server is stable.

So I've reduced allocations in the interpolation as possible. This post describes how I did it and coding tips to avoid allocations.


## 1. Write it correct.

Since it was new feature, I wrote it quickly like writing Python code. (Go can be used like Python or C!).

I've made pull request with "[RFC]" in title. We've discussed about feature and rough design first.


## 2. Write benchmark program.

Before starting tuning, I've prepared benchmark program to confirm performance difference.

Writing benchmark program in Go is easy: Write function like `BenchmarkXxxx(b *testing.B)` in `xxx_test.go`. To see allocations, call `b.ReportAllocs()` in it.

Here is benchmark function I've wrote:

```go
func BenchmarkInterpolation(b *testing.B) {
	mc := &mysqlConn{
		cfg: &config{
			interpolateParams: true,
			loc:               time.UTC,
		},
		maxPacketAllowed: maxPacketSize,
		maxWriteSize:     maxPacketSize - 1,
	}

	args := []driver.Value{
		int64(42424242),
		float64(math.Pi),
		false,
		time.Unix(1423411542, 807015000),
		[]byte("bytes containing special chars ' \" \a \x00"),
		"string containing special chars ' \" \a \x00",
	}
	q := "SELECT ?, ?, ?, ?, ?, ?"

	b.ReportAllocs()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_, err := mc.interpolateParams(q, args)
		if err != nil {
			b.Fatal(err)
		}
	}
}
```

To run it:

```console
$ go test -bench=BenchmarkInterpolation
PASS
BenchmarkInterpolation	  300000	      3887 ns/op	    1144 B/op	      15 allocs/op
ok  	github.com/go-sql-driver/mysql	2.386s
```


## 3. Identify allocations

When setting `allocfreetrace=1` to environment variable `GODEBUG`, you can see stacktrace where allocation occures ([reference](http://golang.org/pkg/runtime/#pkg-overview)).

But using it while running `go test -bench=BenchmarkInterpolation` makes huge log including large noise. To reduce log, build `mysql.test` with `go test -c` and run it.

```console
$ go test -c
$ GODEBUG=allocfreetrace=1 ./mysql.test -test.run=none -test.bench=BenchmarkInter -test.benchtime=10ms 2>trace.log
PASS
BenchmarkInterpolation      5000          4095 ns/op        1144 B/op         15 allocs/op
```

`-test.run=none` prevents running tests before benchmark. `-test.benchtime=10ms` reduces execution time and log size.

Now I have `trace.log`. Open it with vim and search `interpolateParams`. Cut unnecessary stacktrace before it. I can see stacktrace like this:

```
tracealloc(0xc2080100e0, 0x70, string)
goroutine 5 [running]:
runtime.mallocgc(0x70, 0x22e4e0, 0x0, 0x2)
	/usr/local/Cellar/go/1.4.1/libexec/src/runtime/malloc.go:327 +0x32a fp=0xc20802ea60 sp=0xc20802e9b0
runtime.newarray(0x22e4e0, 0x7, 0x15c5e)
	/usr/local/Cellar/go/1.4.1/libexec/src/runtime/malloc.go:365 +0xc1 fp=0xc20802ea98 sp=0xc20802ea60
runtime.makeslice(0x2229c0, 0x7, 0x7, 0x0, 0x0, 0x0)
	/usr/local/Cellar/go/1.4.1/libexec/src/runtime/slice.go:32 +0x15c fp=0xc20802eae0 sp=0xc20802ea98
strings.genSplit(0x30c190, 0x17, 0x2e1f10, 0x1, 0x0, 0x7, 0x0, 0x0, 0x0)
	/usr/local/Cellar/go/1.4.1/libexec/src/strings/strings.go:287 +0x14d fp=0xc20802eb60 sp=0xc20802eae0
strings.Split(0x30c190, 0x17, 0x2e1f10, 0x1, 0x0, 0x0, 0x0)
	/usr/local/Cellar/go/1.4.1/libexec/src/strings/strings.go:325 +0x76 fp=0xc20802ebb0 sp=0xc20802eb60
github.com/go-sql-driver/mysql.(*mysqlConn).interpolateParams(0xc20802eed0, 0x30c190, 0x17, 0xc20802ee70, 0x6, 0x6, 0x0, 0x0, 0x0, 0x0)
	/Users/inada-n/go1.4/src/github.com/go-sql-driver/mysql/connection.go:180 +0x86 fp=0xc20802ed38 sp=0xc20802ebb0
github.com/go-sql-driver/mysql.BenchmarkInterpolation(0xc20806a400)
	/Users/inada-n/go1.4/src/github.com/go-sql-driver/mysql/benchmark_test.go:240 +0x437 fp=0xc20802ef58 sp=0xc20802ed38
testing.(*B).runN(0xc20806a400, 0x1)
	/usr/local/Cellar/go/1.4.1/libexec/src/testing/benchmark.go:124 +0x95 fp=0xc20802ef68 sp=0xc20802ef58
testing.(*B).launch(0xc20806a400)
	/usr/local/Cellar/go/1.4.1/libexec/src/testing/benchmark.go:199 +0x78 fp=0xc20802efd8 sp=0xc20802ef68
runtime.goexit()
	/usr/local/Cellar/go/1.4.1/libexec/src/runtime/asm_amd64.s:2232 +0x1 fp=0xc20802efe0 sp=0xc20802efd8
created by testing.(*B).run
	/usr/local/Cellar/go/1.4.1/libexec/src/testing/benchmark.go:179 +0x3e

...
```

This stacktrace shows `strings.Split()` called from `interpolateParams()` cause allocation. You can see also filename and line number (`connection.go:180`).

Here is the code:

```go
func (mc *mysqlConn) interpolateParams(query string, args []driver.Value) (string, error) {
	chunks := strings.Split(query, "?")
	if len(chunks) != len(args)+1 {
		return "", driver.ErrSkip
	}
```

## 4. Tuning up

OK. It's time to start optimize. Let's record current benchmark result. I use it later.

```console
$ go test -bench=BenchmarkInterpolation | tee old.txt
PASS
BenchmarkInterpolation	  500000	      3942 ns/op	    1144 B/op	      15 allocs/op
ok  	github.com/go-sql-driver/mysql	3.219s
```

### 4.1. Avoid concatenate strings, Use []byte and append

My first version is very Pythonic. When interpolating `db.Exec("SELECT SLEEP(?)", 42)`, my code did like this:

```go
parts := []string{"SELECT SLEEP(", escape(42), ")"}
query := strings.Join(parts, "")
```

Since Go's string is immutable, making temporary string cause allocations. So I replaced `stirngs.Join()` with `[]byte` and `append`. I can use `strconv.AppendInt()` and `strconv.AppendFloat()` to avoid temporal string when formatting int64 and float64.

Now my code looks like:

```go
	buf := make([]byte, 0, estimatedSize)
	argPos := 0

	for _, c := range []byte(query) {
		if c != '?' {
			buf = append(buf, c)
			continue
		}

		arg := args[argPos]
		argPos++

		if arg == nil {
			buf = append(buf, []byte("NULL")...)
			continue
		}

		switch v := arg.(type) {
		case int64:
			buf = strconv.AppendInt(buf, v, 10)
                ...
```

### 4.2. Use benchcmp

I've made first optimization. Let's measure it.

`benchcmp` is famous tool to compare benchmark result. You can include it's output in commit log. It makes your commit looks cool.

```console
$ go get -u golang.org/x/tools/cmd/benchcmp
$ go test -bench=BenchmarkInterpolation > new.txt
$ benchcmp old.txt new.txt
benchmark                  old ns/op     new ns/op     delta
BenchmarkInterpolation     3942          2573          -34.73%

benchmark                  old allocs     new allocs     delta
BenchmarkInterpolation     15             6              -60.00%

benchmark                  old bytes     new bytes     delta
BenchmarkInterpolation     1144          560           -51.05%
```

You can see number of allocation is reduced from 15 to 6.
Execution time and memory consumption are also reduced.


### 4.3. Avoid Time.Format()

`time.Time` doesn't provides methods like `strconv.AppendInt()`.
To avoid making temporary string, I've wrote it myself.

Before:

```go
		case time.Time:
			if v.IsZero() {
				buf = append(buf, []byte("'0000-00-00'")...)
			} else {
				fmt := "'2006-01-02 15:04:05.999999'"
				if v.Nanosecond() == 0 {
					fmt = "'2006-01-02 15:04:05'"
				}
				s := v.In(mc.cfg.loc).Format(fmt)
				buf = append(buf, []byte(s)...)
			}
```

After (including other improvements):

```go
		case time.Time:
			if v.IsZero() {
				buf = append(buf, "'0000-00-00'"...)
			} else {
				v := v.In(mc.cfg.loc)
				v = v.Add(time.Nanosecond * 500) // To round under microsecond
				year := v.Year()
				year100 := year / 100
				year1 := year % 100
				month := v.Month()
				day := v.Day()
				hour := v.Hour()
				minute := v.Minute()
				second := v.Second()
				micro := v.Nanosecond() / 1000

				buf = append(buf, []byte{
					'\'',
					digits10[year100], digits01[year100],
					digits10[year1], digits01[year1],
					'-',
					digits10[month], digits01[month],
					'-',
					digits10[day], digits01[day],
					' ',
					digits10[hour], digits01[hour],
					':',
					digits10[minute], digits01[minute],
					':',
					digits10[second], digits01[second],
				}...)

				if micro != 0 {
					micro10000 := micro / 10000
					micro100 := micro / 100 % 100
					micro1 := micro % 100
					buf = append(buf, []byte{
						'.',
						digits10[micro10000], digits01[micro10000],
						digits10[micro100], digits01[micro100],
						digits10[micro1], digits01[micro1],
					}...)
				}
				buf = append(buf, '\'')
			}
```

It reduces two allocations:

```
    benchmark                  old allocs     new allocs     delta
    BenchmarkInterpolation     6              4              -33.33%
```


### 4.4. Avoid `range` when iterating string

Usually, `range` is used for iterating slice. But `for _, c := range s` (where `s` is string) produces `rune`, not `byte`.

My first code used `for i, c := range([]byte(s)) {`. But Go 1.4 make new slice and copy `s` to it. (Go 1.5 optimize it out).

So I've used C-like for loop:

```
@@ -210,8 +210,8 @@ func (mc *mysqlConn) interpolateParams(query string, args []driver.Value) (strin
        buf := make([]byte, 0, estimated)
        argPos := 0

-       // Go 1.5 will optimize range([]byte(string)) to skip allocation.
-       for _, c := range []byte(query) {
+       for i := 0; i < len(query); i++ {
+               c := query[i]
                if c != '?' {
```

It reduces one allocation.

```
    benchmark                  old allocs     new allocs     delta
    BenchmarkInterpolation     4              3              -25.00%
```


### 4.5. `append` `string` to `[]byte` directly

Concatenating slices can be written like `buf = append(buf, token...)`. Basically, `buf` and `token` should have same type. But there is one exception: when `buf` is `[]byte`, `token` can be `string`.

```
@@ -210,17 +210,19 @@ func (mc *mysqlConn) interpolateParams(query string, args []driver.Value) (strin
        argPos := 0

        for i := 0; i < len(query); i++ {
-               c := query[i]
-               if c != '?' {
-                       buf = append(buf, c)
-                       continue
+               q := strings.IndexByte(query[i:], '?')
+               if q == -1 {
+                       buf = append(buf, query[i:]...)
+                       break
                }
+               buf = append(buf, query[i:i+q]...)
+               i += q

                arg := args[argPos]
                argPos++

                if arg == nil {
-                       buf = append(buf, []byte("NULL")...)
+                       buf = append(buf, "NULL"...)
                        continue
                }
...
```

I couldn't reduce allocation from this change. But I could improve readability a lot.


### 4.6. Make two same function: for []byte and for string

MySQL requires same escaping for binary and string.

Sadly, since Go doesn't have "read only slice", converting between `string` and `[]byte` cause allocation and copy. Current compiler can optimize it only for very simple cases.

So I've made mostly same two functions for `[]byte` and `string`.
Generally speaking, duplicated code is evil. But `TEXT` and `BLOB` column may store large data. I want to avoid allocating and copying large data. So I did evil.

It reduces one more allocation.

```
    benchmark                  old ns/op     new ns/op     delta
    BenchmarkInterpolation     2463          2118          -14.01%

    benchmark                  old allocs     new allocs     delta
    BenchmarkInterpolation     3              2              -33.33%

    benchmark                  old bytes     new bytes     delta
    BenchmarkInterpolation     496           448           -9.68%
```

### 4.7. Use send/receive buffer for scratchpad

Last two allocation is: (1) Make scratchpad buffer for interpolate, and (2) convert it to string.

I can remove them by create MySQL packet directly. But it may breaks separation of packet layer and higher layer. So I didn't it for now.

Instead, I borrowed send/receive buffer and use it as scratchpad buffer. It reduces one allocation.

```
    benchmark                  old ns/op     new ns/op     delta
    BenchmarkInterpolation     1900          1403          -26.16%

    benchmark                  old allocs     new allocs     delta
    BenchmarkInterpolation     2              1              -50.00%

    benchmark                  old bytes     new bytes     delta
    BenchmarkInterpolation     448           160           -64.29%
```

## Conclusion

Go's compiler is not intelligent compared with gcc or clang. You should use some tips to reduce allocation.

But Go's runtime is great. I can measure and find allocation quickly. I think it is more important than magical optimization.
