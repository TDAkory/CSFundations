# [FNV hash](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function)

- [FNV Hash](http://www.isthe.com/chongo/tech/comp/fnv/)

> FNV hashes are designed to be fast while maintaining a low collision rate. The FNV speed allows one to quickly hash lots of data while maintaining a reasonable collision rate. The high dispersion of the FNV hashes makes them well suited for hashing nearly identical strings such as URLs, hostnames, filenames, text, IP addresses, etc.

The core of the `FNV-1` hash algorithm is as follows:

```shell
hash = offset_basis
for each octet_of_data to be hashed
 hash = hash * FNV_prime
 hash = hash xor octet_of_data
return hash
```

There is a minor variation of the `FNV` hash algorithm known as `FNV-1a`:
```shell
hash = offset_basis
for each octet_of_data to be hashed
 hash = hash xor octet_of_data
 hash = hash * FNV_prime
return hash
```

The only difference between the `FNV-1a` hash and the `FNV-1` hash is the order of the xor and multiply. The `FNV-1a` hash uses the same FNV_prime and offset_basis as the `FNV-1` hash of the same n-bit size.
