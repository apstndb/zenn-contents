---
title: "Is Spanner Really That Expensive? The Surprising Break-Even Point with Firestore"
published: true
description: "A detailed cost analysis reveals that Spanner can be significantly cheaper than Firestore much earlier than you might think. Let's debunk the myth of Spanner's high cost with concrete numbers."
tags: ["gcp", "spanner", "firestore", "cost"]
---

***Note from the author:*** *This article was originally written in Japanese for the community in Japan. The original version can be found [here](https://zenn.dev/apstndb/articles/spanner-cost-comparison-firestore). Believing the content could be valuable to a global audience, I've translated it into English. The translation was performed with the assistance of Google's Gemini 2.5 Pro. I have reviewed and edited the translation for accuracy and clarity.*

A recent article on the tech blog of **KAUCHE, a Japanese e-commerce company,** "[Firestore → Cloud Spanner DB Cost Reduction of 93%! The Complete Record of a Year-Long Zero-Downtime Migration](https://zenn.dev/kauche/articles/1e733da3748ee1)" (in Japanese), has been making waves.
Anyone who has used Firestore (or its predecessor, Datastore) knows that beyond a certain usage level, it can become more expensive than Spanner. However, database migrations are notoriously difficult, so many users likely remain on Firestore despite the cost. The fact that KAUCHE accomplished a zero-downtime migration from Firestore to Spanner is truly impressive.

Interestingly, looking at the reactions to this article, I noticed a common thread of surprise that migrating from Firestore to Spanner could actually *reduce* costs.

https://x.com/search?q=https%3A%2F%2Fzenn.dev%2Fkauche%2Farticles%2F1e733da3748ee1&src=typed_query&f=live

Why does this surprise exist, when there are options within Google Cloud like Firestore that can become significantly more expensive than Spanner? I believe it's because Firestore offers a generous daily free tier, making it easy to start small, while Spanner has no free tier and is plagued by a persistent preconception that it is "expensive."

This misconception can hinder optimal technology choices, leading to lost opportunities for developers, businesses, and even Google. It can cause tangible financial harm and create significant technical debt by forcing a difficult database migration in the future.

Google itself has been trying to dispel this myth, publishing articles like "[Cloud Spanner myths busted](https://cloud.google.com/blog/products/databases/cloud-spanner-myths-busted)" and holding sessions like the "[Spanner: Database Unlimited Series](https://www.youtube.com/playlist?list=PLIivdWyY5sqJPSoX2R4mRq_wyg0JTjrAG)" based on it.

> **Why did this "expensive" image stick?**
> As Google's article mentions, it's likely due to outdated impressions:
> - **High minimum cost for production instances with an SLA:** It used to require a minimum of 3 nodes, costing around $2,000/month even in the cheapest `us-central1` region.
>     - In [September 2019](https://cloud.google.com/spanner/docs/release-notes#September_25_2019), this was reduced to 1 node with an SLA, cutting the cost by two-thirds.
>     - With granular instance sizing ([GA in May 2022](https://cloud.google.com/spanner/docs/release-notes#May_31_2022)), it became possible to get an SLA with just 0.1 nodes (100 PUs), reducing the cost by another factor of 10.
>     - The cost is now comparable to a Cloud SQL `db-g1-small` instance with an HA configuration.
> - **No free development environment:** Since it's not open-source, development environments incurred similar costs.
>     - This has been addressed with the official [Spanner emulator](https://cloud.google.com/spanner/docs/emulator) ([GA in July 2020](https://cloud.google.com/spanner/docs/release-notes#July_30_2020)) and [free trial instances](https://cloud.google.com/spanner/docs/free-trial-instance) ([GA in September 2022](https://cloud.google.com/spanner/docs/release-notes#September_08_2022)).

Despite these efforts, the message may not have fully resonated without concrete numbers. And comparing it to competitor products is often frowned upon.

So, I thought, why not do a quantitative comparison between Google Cloud's own database offerings? There seems to be value in making that public.

## Firestore vs. Spanner: A Price Comparison

In this article, I'll perform a price comparison between Firestore and Spanner, inspired by the cost issues highlighted by KAUCHE's experience.

> **A few notes on the comparison:**
> - This is a simplified comparison focusing only on Firestore and Spanner among Google Cloud's database offerings.
>   - I'll assume Firestore is used as a backend database. The differences between Native mode and Datastore mode are not critical to the conclusion.
>   - I will not compare them with services from other cloud providers.
>   - This is not a general SQL vs. NoSQL or SQL vs. Distributed SQL comparison.
> - The comparison is based solely on information from the official documentation.
>   - No load testing is involved.
> - Since a key goal is to raise awareness, I will be direct about the cost challenges of Firestore, based on the data.

### Extracting Relevant Values from Official Docs

- All values are as of July 9, 2025.
- The region is standardized to `asia-northeast1` (Tokyo).
- Currency is kept in USD to avoid exchange rate fluctuations.
- Committed Use Discounts (CUDs) are omitted as they are the same for both Firestore and Spanner (20% for 1 year, 40% for 3 years) and don't affect the relative comparison.
- Network egress fees are the same for both and are not included in this comparison's scope, though they can affect the total cost.

#### Firestore

Firestore is entirely pay-as-you-go, so all the pricing information is on one page.

- [Firestore pricing](https://cloud.google.com/firestore/pricing?hl=en)
- [Firestore in Datastore mode pricing](https://cloud.google.com/datastore/pricing?hl=en)

Native mode and Datastore mode have slightly different terminology but the same pricing structure. I'll use the newer Native mode terms.

| Firestore Native mode | Datastore mode     | Free usage     | Cost                                  |
|-----------------------|--------------------|----------------|---------------------------------------|
| Document reads        | Entity reads       | 50,000 per day | $0.038 per 100,000 entities/documents |
| Document writes       | Entity writes      | 20,000 per day | $0.115 per 100,000 entities/documents |
| Document/TTL deletes  | Entity/TTL deletes | 20,000 per day | $0.013 per 100,000 entities/documents |
| Stored data           | Stored data        | 1 GiB          | $0.115/GiB/month                      |
| PITR data             | PITR data          | N/A            | $0.115/GiB/month                      |
| Backup data           | Backup data        | N/A            | $0.038/GiB/month                      |
| Restore operation     | Restore operation  | N/A            | $0.256/GiB                            |
| N/A                   | Small operations   | 50,000 per day | Free                                  |

> Note: Firestore Native mode doesn't have "small operations." Index entry reads and aggregation queries are counted as one document read per 1,000 entries. Also, the [Firestore editions overview](https://cloud.google.com/firestore/native/docs/editions-overview) states that the Standard edition uses Hybrid storage (SSD & HDD).

#### Spanner

Spanner is billed based on provisioned capacity in Processing Units (PUs). We need to look at two pages:

- [Spanner pricing](https://cloud.google.com/spanner/pricing)
- [Performance overview](https://cloud.google.com/spanner/docs/performance)

##### Compute Capacity

| Edition         | Cost per node (including all three replicas) |
|-----------------|----------------------------------------------|
| Standard        | **$1.17 per hour**                           |
| Enterprise      | $1.599 per hour                              |
| Enterprise Plus | $2.223 per hour                              |

The features in the Standard edition are sufficient for this comparison, so I'll focus on that.

##### Database Storage

| Storage Type | Cost per GB (including all three replicas) |
|--------------|--------------------------------------------|
| SSD          | **$0.39 per GB per month**                 |
| HDD          | $0.078 per GB per month                    |

I'll focus on SSD storage for this comparison.

##### Estimated Throughput from Docs

While database performance typically requires workload-specific benchmarks, Spanner provides estimated throughput for simple workloads in its documentation. This is a reasonable approximation for a comparison with Firestore, which is primarily used for key-based reads and writes without complex queries.

> Read guidance is given per region (because reads can be served from any read-write or read-only region), while write guidance is for the entire configuration. Read guidance assumes you're reading single rows of 1KB. Write guidance assumes that you're writing single rows at 1KB of data per row.

Here is the estimated throughput for a regional configuration like `asia-northeast1`:

> [Performance for regional configurations](https://cloud.google.com/spanner/docs/performance#regional-performance)
> Each 1,000 processing units (1 node) of compute capacity can provide the following peak performance (at 100% CPU) in a regional instance configuration:

| Peak reads (QPS per region)     | Peak writes (QPS total)           |
|:--------------------------------|:----------------------------------|
| **SSD: 22,500** <br> HDD: 1,500 | or **SSD: 3,500** <br> HDD: 3,500 |

I'll use the highlighted SSD values.

> **(Note)** This catalog spec has improved significantly. An announcement on [October 11, 2023](https://cloud.google.com/spanner/docs/release-notes#October_11_2023) detailed [Performance and storage improvements](https://cloud.google.com/spanner/docs/performance#improved-performance), boosting typical throughput by 1.5x for both reads and writes and increasing the storage limit by 2.5x. If you want to verify these numbers, you can check the YCSB benchmark results in "[Benchmarking Spanner's cost-performance for key-value workloads](https://cloud.google.com/blog/products/databases/benchmarking-spanner-for-key-value-workloads)".

##### Spanner's Minimum Configuration: Price and Throughput

A Spanner instance can be created with as little as 100 Processing Units (PUs), which is 0.1 nodes. This configuration is production-ready and comes with an SLA. Here are the adjusted numbers for this minimum setup:

| Item                           | Value           |
|--------------------------------|-----------------|
| Standard edition cost (100 PU) | $0.117 per hour |
| Peak reads (QPS per region)    | 2,250           |
| Peak writes (QPS total)        | 350             |

This is at 100% CPU utilization. However, the official documentation ([Alerts for high CPU utilization](https://cloud.google.com/spanner/docs/cpu-utilization#recommended-max)) recommends keeping CPU utilization for high-priority tasks below 65% for regional instances. I will factor this into the comparison.

> **Why 65%?** While not explicitly stated, this is likely to maintain service levels during a zonal failure. A regional Spanner instance has R/W replicas in three zones. If one zone goes down, you lose 1/3 of your capacity. To handle the same load, the remaining 2/3 of capacity (approx. 67%) must suffice. Staying at 65% ensures that a zone failure won't push CPU usage over 100%. This logic also explains the 45% recommendation for dual/multi-region instances, which have replicas in two R/W regions (losing one leaves 50% capacity).

### The Comparison

Let's find the break-even point between the minimum Spanner configuration and Firestore.

- I'll assume a steady workload of either 1KB reads only or 1KB writes only.
- I'll assume the schema is well-designed to achieve theoretical performance.
- I'll include Firestore's daily free tier.
- I'll factor in Spanner's 65% recommended CPU utilization.

The break-even point is where `Firestore Cost = Spanner Cost`. We can find the number of daily operations, `x`, with the formula: {% katex %}
x = \frac{C_{\text{spanner}}}{P_{\text{firestore}}} + F_{\text{free}}
{% endkatex %}

**Variables:**
- {% katex inline %}x{% endkatex %} = Number of operations per day (what we want to find)
- {% katex inline %}F_{\text{free}}{% endkatex %} = Firestore free tier operations (Reads: 50,000, Writes: 20,000)
- {% katex inline %}P_{\text{firestore}}{% endkatex %} = Firestore price per operation (Read: \$0.00000038/op, Write: \$0.00000115/op)
- {% katex inline %}C_{\text{spanner}}{% endkatex %} = Spanner daily cost (100 PU = \$0.117/hr * 24h = \$2.808/day)

<div>
  <p><b>To be more precise:</b> The above formula is a simplification. To be perfectly accurate, you need to account for Firestore's 100,000-operation billing increments. The break-even point <code>x</code> is more accurately calculated with the following formula:</p>
  {% katex %}
  x = F_{\text{free}} + \lceil\frac{C_{\text{spanner}}/P_{\text{firestore}}}{100,000}\rceil \times 100,000
  {% endkatex %}
</div>

Here are the results, visualized with graphs. The plots are continuous approximations for simplicity, but the break-even point calculations are precise.

#### Reads

The break-even point for reads is **7,450,000 reads/day**, which is about **86 reads/sec**.
This is only **3.8%** of the peak performance of a 100 PU Spanner instance.

![A line graph comparing the daily cost of Firestore versus Spanner for read operations. The x-axis shows daily reads and the y-axis shows daily cost in USD. The Firestore cost line rises steeply, while the Spanner cost line is flat, showing Spanner is much cheaper at high read volumes.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/md33be8qyxnrq8k3or4r.png)

If you utilize Spanner up to the recommended 65% of its theoretical throughput, it becomes **17 times cheaper** than Firestore. This clearly shows how sticking with Firestore at scale can lead to massive cost increases.

#### Writes

The break-even point for writes is **2,520,000 writes/day**, which is about **29 writes/sec**.
This is about **8.1%** of the peak performance of a 100 PU Spanner instance.

![A line graph comparing the daily cost of Firestore versus Spanner for write operations. The x-axis shows daily writes and the y-axis shows daily cost in USD. The Firestore cost line rises steeply, while the Spanner cost line is flat, demonstrating Spanner's cost-effectiveness at high write volumes.](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c82gs8o49mi03skmx6e3.png)

If you utilize Spanner up to the recommended 65% of its throughput, it is **8 times cheaper** than Firestore.

#### Storage

A direct storage cost comparison is difficult because Firestore uses hybrid storage. However, we can get a rough idea. A 100 PU Spanner instance has a storage limit of 1024 GiB.

| Service   | Unit Price (per GiB per month) | Monthly Price (for 1024 GiB) |
|-----------|--------------------------------|------------------------------|
| Spanner   | $0.39                          | $399.36                      |
| Firestore | $0.115                         | $117.76                      |
| Ratio     |                                | 339%                         |
| Diff      |                                | $281.6                       |

While Spanner's storage unit price is higher, the difference is not the dominant factor. In a scenario where you are maxing out the read/write capacity of a 100 PU instance, the daily cost difference for operations is in the tens of dollars, which far outweighs the storage cost difference.

### Analysis

The calculations show that while Spanner isn't always cheaper, it becomes significantly more cost-effective than Firestore beyond a certain scale. This isn't surprising. Both databases offer strong consistency and high availability through geographic distribution, making them comparable services. The main differences are the data model and the billing model.

Generally, provisioned capacity is discounted relative to on-demand, pay-per-operation pricing.

> In fact, Firestore and Spanner are more than just similar. It has been suggested since Firestore's launch, and later confirmed in Google research papers, that **Firestore is implemented on top of Spanner**. The architecture of Firestore is detailed in the **"[Firestore: The NoSQL Serverless Database for the Application Developer](https://research.google.com/pubs/firestore-the-nosql-serverless-database-for-the-application-developer/)"** paper, and its implementation on Spanner is further described in **"[Transparent Migration of Datastore to Firestore](https://research.google.com/pubs/transparent-migration-of-datastore-to-firestore/)"**.
>
> In other words, Firestore is a high-value managed service that abstracts away complexities like query planning and capacity management by offering Spanner's powerful infrastructure on a multi-tenant basis, which provides an easy, free-to-start experience. The price difference at scale can be seen as the cost of this abstraction.

## To Anyone Considering Firestore

If you're considering Firestore for your backend, keep these points in mind:

- Firestore's cost advantage is limited to a relatively narrow range for a horizontally scaling database: from the free tier up to a few million operations per day.
- Be aware that costs will scale with `number of users × operations per user`.
- If your service has a fixed number of users (e.g., an internal tool), Firestore is likely a safe choice. Otherwise, always evaluate the possibility that Spanner will become cheaper.
- If your service can justify the monthly cost of a 100 PU Spanner instance (approx. $85/month), choosing Spanner from the start might give you more peace of mind.

> For real-time database use cases where clients connect directly, Firestore is often the right choice. Building the real-time capabilities yourself on top of Spanner would be a massive undertaking. In that case, the comparison should be against services like Firebase Realtime Database.

## Conclusion

Firestore is often perceived as a cheap database, suitable for personal and small-scale projects. However, this analysis shows that the cost break-even point with Spanner's minimum configuration (approx. $85/month) is reached at less than 10% of that Spanner instance's capacity.

Beyond that point, Spanner becomes dramatically cheaper: up to **17 times cheaper for reads** and **8 times cheaper for writes**. An $85/month cost is not significantly higher than the minimum cost for other HA database configurations on Google Cloud.

So, is Spanner still "expensive"? If you hear someone say that, ask them under what conditions, compared to what, and which specific SKU they are talking about.

## Appendix: Firestore Enterprise Edition

For those who think Firestore becomes too expensive, you might want to look at the new Firestore Enterprise edition, announced at Cloud Next 2025 and currently in preview.

> While it currently only offers MongoDB compatibility, the [Firestore editions overview](https://cloud.google.com/firestore/native/docs/editions-overview) states that Native mode support is planned for the future.

According to the [Firestore Enterprise edition pricing](https://cloud.google.com/firestore/enterprise/pricing), it has a pricing structure geared towards larger-scale users compared to the Standard edition.

|                        | Standard edition             | Enterprise edition          | Ratio (Enterprise/Standard) |
|------------------------|------------------------------|-----------------------------|-----------------------------|
| Read Units             | $0.038 per 100,000 docs      | $0.0642 per 1,000,000 units | 16.9 %                      |
| Write Units            | $0.115 per 100,000 docs      | $0.3339 per 1,000,000 units | 29 %                        |
| Stored Data            | $0.115/GiB/month             | $0.312/GiB/month            | 271 %                       |

Note the change in billing unit from 100K to 1M operations. This makes read/write operations cheaper for small documents. However, units are calculated in 4 KiB increments for reads and 1 KiB for writes. This means documents larger than 24KB (for reads) or 4KB (for writes) could become more expensive than in the Standard edition.

Even with this new pricing, the fundamental trend of becoming more expensive than Spanner at scale seems to persist. It would be ideal if users who start small with Firestore didn't have to regret their choice later. A future where Firestore offers a provisioned capacity model, similar to Spanner, would be a welcome development, combining the ease of getting started with the cost-efficiency of Spanner at scale.