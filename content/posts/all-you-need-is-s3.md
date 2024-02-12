---
title: "All You Need Is S3"
date: 2024-02-12T12:35:44+01:00
draft: false
---

By: [Damian Peckett](mailto:damian@pecke.tt)

Over the past decade, the tech industry has significantly invested in the development of the cloud-native ecosystem, pouring vast amounts of human capital into its expansion. Despite the apparent diversity, a common underlying challenge unifies most projects: achieving consistent data sharing between processes.

The term "data" encompasses a broad spectrum, from files and messages to database records and event streams. The current dominant paradigm has been to partition the task of sharing data into distinct domains, such as object storage, message queues, and databases. These domain specific systems invariably feature a half baked implementation of Paxos <sup>\[1\]\[2\]</sup>, a custom persistence layer, and bespoke APIs.

However, at its core, data remains merely a sequence of bytes. Do we really need to slice it, dice it, and treat it so differently?

Admittedly, each system is optimized for specific data types, employing various optimizations and transformations (e.g., indexing, compression, etc) before persistence. But when it comes time to flush the data to disk, the target is invariably a file sitting on a filesystem. So it's pretty clear that we all implicitly agree that when it comes to durabilitiy, POSIX filesystems are "good" enough <sup>\[3\]</sup>.

Less well known by most developers is that POSIX included more than just regular files. It also included synchronization primitives such as file locks <sup>\[4\]\[5\]\[6\]</sup>. File locking allows multiple processes to synchronize shared access to a file. Wait a minute, remember the initial problem the cloud-native ecosystem set out to address? POSIX has provided a solution on single-node systems for decades, a solution that has become so integral it often goes unnoticed.

But can the concept of files extend beyond a single machine? Various network filesystems have tried, often sacrificing consistency for performance <sup>\[7\]</sup>. Yet, CephFS's success and IBM Spectrum Scale (GPFS) indicate that sacrificing consistency isn't always necessary.

The most interesting in this space is Google's Colossus<sup>\[8\]</sup>, the network filesystem at the heart of Google's cloud storage. It doesn't strictly adhere to POSIX; instead, it uses an in-process client library for filesystem communication, allowing applications to adjust consistency as needed.

Interestingly, AWS initially lacked a globally consistent network filesystem, and instead promoted an eventually consistent object storage model. Designing a performant network filesystem is fiendishly difficult and if you just give up on the idea of strong consistency there's a lot of "free" performance to be had.

But the problem is when your storage layer is eventually consistent, you have to build a lot of extra supporting infrastructure to make up for it. While eventually consistent object storage is cheap, the supporting infrastructure is not.

My unconvential take is that what if we just used files on a network filesystem as the underlying cloud infrastructure abstraction? Seriously what's the difference between a file and a document in a key-value store? What if a message queue was just a directory of sorted files? A lot of the boring infrastructure we use today could be replaced with simple abstraction libraries.

Admittedly there are some performance concerns but I think most of them can be addressed by adopting a flexible consistency model akin to that of Google's Colossus.

I think the main reason that this hasn't happened already is that it took the cloud providers a long time to offer good network filesystem products, and by the time they had the ecosystem had already embraced the eventual consistency model. But I think it's overdue for us to re-evaluate the tradeoffs we've made.

Returning to S3, while originally it was an eventually consistent object store, it has evolved to adopt many filesystem-like semantics <sup>\[9\]\[10\]</sup>, and in recent years has even offered strong read-after-write consistency <sup>\[11\]</sup>. The way I look at it is that such a network filesystem product already exists, it's just not POSIX. It's S3. 

Forgive me for the advertisement but after growing frustrated with clunky vendor S3 UIs and how difficult it is to share buckets with non-technical users I've developed [Bucketeer](https://github.com/bucket-sailor/bucketeer) which I'm hoping to grow into the "Ultimate S3 Bucket Explorer". I highly recommend you check it out!

## References

1. L. Lamport, "Paxos Made Simple," ACM SIGACT News (Distributed Computing Column) 32, no. 4 (Whole Number 121, December 2001), pp. 51-58, December 2001. Available: https://www.microsoft.com/en-us/research/publication/paxos-made-simple/.
2. Jepsen, "Analyses," Jepsen.io, 2023. Available: https://jepsen.io/analyses.
3. A. Aghayev, S. Weil, M. Kuchnik, M. Nelson, G. R. Ganger, and G. Amvrosiadis, "File systems unfit as distributed storage backends: lessons from 10 years of Ceph evolution," in Proceedings of the 27th ACM Symposium on Operating Systems Principles (SOSP '19), Huntsville, Ontario, Canada, 2019, pp. 353-369. Available: https://dl.acm.org/doi/abs/10.1145/3341301.3359656. doi: 10.1145/3341301.3359656.
4. L. Poettering, "On the Brokenness of File Locking," 0pointer.de, June 26, 2010. Available: https://0pointer.de/blog/projects/locking.html.
5. J. T. Layton, "File-private POSIX locks," LWN.net, February 19, 2014. Available: https://lwn.net/Articles/586904/.
6. "The GNU C Library Manual," section "Open File Description Locks," Free Software Foundation. Available: https://www.gnu.org/software/libc/manual/html_mono/libc.html#Open-File-Description-Locks.
7. O. Kirch. Why NFS Sucks. Proceedings of the
2006 Ottawa Linux Symposium, Ottawa, Canada,
July, 2006. Available: https://www.kernel.org/doc/ols/2006/ols2006v2-pages-59-72.pdf.
8. "Colossus under the hood: a peek into Google’s scalable storage system," Google Cloud Blog, 20 April 2021. Available: https://cloud.google.com/blog/products/storage-data-transfer/a-peek-behind-colossus-googles-file-system.
9. "ListObjectsV2 - Amazon Simple Storage Service," Amazon Web Services, Inc., 2023. Available: https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html. 
10. "Uploading and copying objects using multipart upload - Amazon Simple Storage Service," Amazon Web Services, Inc., 2023.  Available: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html.
11. J. Barr, "Amazon S3 Update – Strong Read-After-Write Consistency," AWS News Blog, 01-Dec-2020. Available: https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/.