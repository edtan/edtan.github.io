---
title: Restoring libcrypto on remote host
description: How to restore libcrypto (or any other shared libraries) after accidentally deleting it on a remote server
---

Recently, I ran into a situation where I was trying to update openssl on
a CentOS 7 server, but accidentally overwrote the existing version of
libcrypto.so with the new version.  I was lucky that there were other
similar servers that had the same previous version of libcrypto.so
available, so I attempted to scp it over.  However, this failed because
scp was now broken as it was trying to reference the old libcrypto.so.
wget and `yum reinstall openssl` failed with the same underlying issue.
I was in a bit of a panic because I could not make any new SSH
connections to the server, and only had my existing SSH connection to
rely on.  Luckily, someone had ran into the same situation before and
had it answered in this [StackOverflow
post](https://serverfault.com/a/587368/578155) - the solution was to
encode the shared library into base64 and copy it over using the
clipboard!

As for the reason I overwrote libcrypto.so - it was a stupid mistake on
my part regarding symbolic links.  As a result of building openssl,
it produced a new shared library libcrypto.so.1.1 and a corresponding
symbolic link libcrypto.so pointing to it:

```
libcrypto.so -> libcrypto.so.1.1
```

I wanted to update the symbolic link at /lib64/libcrypto.so (->
libcrypto.so.1.0.2k) to point to this new version, so I thought I could
simply "replace" the symbolic link by running:

```
sudo cp libcrypto.so /lib64/libcrypto.so
```

However, because these are symbolic links, this ends up copying
libcrypto.so.1.1 to /lib64/libcrypto.so.1.0.2k, and I got a
`segmentation fault` error after running that command.  It seems that I
should have run something like:

```
ln -sf $(readlink libcrypto.so) /lib64/libcrypto.so
```

Finally, the reason I was trying to update openssl was to update the
version of libssl used by Python 2.7 in CentOS 7.  It seems that Python
is linked to the previous version of libssl, because even after
installing the new openssl (including after both running ldconfig
manulaly, as well as running `make install`), the following still
returned the old version:

```
python -c 'import ssl; print(ssl.OPENSSL_VERSION)'
# OpenSSL 1.0.2k-fips  26 Jan 2017 
```

Additionally, I used strace to see which versions were loaded when
trying to use the requests library, and it showed that after opening
/etc/ld.so.cache, Python then opened the old /lib64/libssl.so.10 and
/lib64/libcrypto.so.10.  Thus, Python would either need to be updated or
recompiled to link to a newer version of openssl; both which didn't seem
like a good idea, as I'm not sure how it would affect CentOS 7.  In the
end, we did not need to use an updated version of libssl.
