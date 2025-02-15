# valkeylock

A [Valkey Distributed Lock Pattern](https://redis.io/docs/reference/patterns/distributed-locks/) enhanced by [Client Side Caching](https://redis.io/docs/manual/client-side-caching/).

```go
package main

import (
	"context"
	"github.com/valkey-io/valkey-go"
	"github.com/valkey-io/valkey-go/valkeylock"
)

func main() {
	locker, err := valkeylock.NewLocker(valkeylock.LockerOption{
		ClientOption: valkey.ClientOption{InitAddress: []string{"node1:6379", "node2:6380", "node3:6379"}},
		KeyMajority:  2, // please make sure that all your `Locker`s share the same KeyMajority
	})
	if err != nil {
		panic(err)
	}
	defer locker.Close()

	// acquire the lock "my_lock"
	ctx, cancel, err := locker.WithContext(context.Background(), "my_lock")
	if err != nil {
		panic(err)
	}

	// "my_lock" is acquired. use the ctx as normal.
	doSomething(ctx)

	// invoke cancel() to release the lock.
	cancel()
}
```

## Features backed by the Valkey Client Side Caching
* The returned `ctx` will be canceled automatically and immediately once the `KeyMajority` is not held anymore, for example:
  * Valkey down.
  * Related keys has been deleted by other program or administrator.
* The waiting `Locker.WithContext` will try acquiring the lock again automatically and immediately once it has been released by someone even from other program.

## How it works

When the `locker.WithContext` is invoked, it will:

1. Try acquiring 3 keys (given that `KeyMajority` is 2), which are `valkeylock:0:my_lock`, `valkeylock:1:my_lock` and `valkeylock:2:my_lock`, by sending valkey command `SET NX PXAT` or `SET NX PX` if `FallbackSETPX` is set.
2. If the `KeyMajority` is satisfied within the `KeyValidity` duration, the invocation is successful and a `ctx` is returned as the lock.
3. If the invocation is not successful, it will wait for client-side caching notification to retry again.
4. If the invocation is successful, the `Locker` will extend the `ctx` validity periodically and also watch client-side caching notification for canceling the `ctx` if the `KeyMajority` is not held anymore.

### Disable Client Side Caching

Some Valkey provider doesn't support client-side caching, ex. Google Cloud Memorystore.
You can disable client-side caching by setting `ClientOption.DisableCache` to `true`.
Please note that when the client-side caching is disabled, valkeylock will only try to re-acquire locks for every ExtendInterval.