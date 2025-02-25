- 50 years of SQL — [[October 27th, 2024]]
    - To each of the papers, the previous papers are kind of like a "test of time" award recipient, a seminal paper from a previous technological era.
    - Also, computers got 32x faster every decade. So that's kind of amazing. Cost per gigaflop went from $15,000,000,000 to $0.30, yet the SQL language work remains the same, and its intended use case is just as salient now than ever.
    - SEQUEL: …language or menu selection (3,4) seem to be the most viable alternatives. However, there is also a large class of users who, while they are not computer specialists, would be willing to learn to interact with a computer in a reasonably high-level, non-procedural query language. Examples of such users are accountants, engineers, architects, and urban planners. It is for this class of users that SEQUEL is intended.
    - Compare to the predicate calculus, SQL tries to be more readable. Needs: Easy-to-maintain, linear notation format.
        - This is interesting, reminds me of how computing interface ideas are more timeless than you might think. There's not so much complexity that can be supported in an interface. Humans are simple, and even kids can use an iPhone.
        - SQL is a tool, and it connects the user with the compute. The nature of people hasn't changed, and neither has the fundamental nature of computation.
        - Programming languages have evolved over decades to be this masterpiece that allows us to reason about and express computational ideas.
        - Composition operator makes sense, no lowercase letters. Invented the word "join."
    - Now there are 14 people. We're getting on to the second paper, a Critique of SQL.
        - Very bolded text, lol: "The criticisms that follow should not be construed as criticisms of the original designers and implementers of the SQL language. The paper is intended solely as a critique of the SQL language as such, and nothing more."
        - An old culture of criticizing the idea, rather than the person behind the idea.
        - Lol this should be required reading for anyone who doesn't like SQL, they're super matter-of-fact about the lack of constructors / data model clarity / assignments. ![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2Fekzhang%2FCxgidsVs-V.png?alt=media&token=84ba4a3c-5cfd-45fd-8867-69d39a1d12bf)
        - But SQL lives on, and these other languages do not. So I mean, what went wrong? Or what did SQL do right that this analysis cannot capture?
        - They correctly observe that users might have to transform their query to be "less natural" when they add a level of nesting. It's not a very composable language like others. Maybe `NYC UNION SFO` should be valid, just as `SELECT * FROM NYC UNION SELECT * FROM SFO` — but in practice, the latter is "good enough."
        - PRQL-lang.org would have a field day with this, orthogonal language features.
        - Good points about COUNT and aggregates being outside, though being inside allows the groupby-count workflow to be simpler, which is perhaps more common! I ran into this awkwardness when designing https://percival.ink as well.
        - "SQL does not exploit the exception-handling capabilities of the host" LOL this predates the idea of client-server databases.
        - Apparently SQL has no foreign key support? I guess database implementations added that. It that could be a boon though, since it made OLAP flows more flexible.
        - It's fascinating to see how much criticism is just "I don't like SELECT * FROM" or something since it's error-prone. At the end of the day, like any programming language, SQL is a compromise: between declarative and imperative, interactive and application-embedded, transactional and analytic, simple and complex, English-like and composable.
    - Critique of ANSI SQL Isolation Levels
        - First "modern" database conference paper of the bunch. Invented snapshot isolation, characterized the levels in terms of forbidden phenomena. Dirty reads, non-repeatable reads, and phantoms.
        - Important is the distinction between RR and SI. In repeatable read, you can have phantom reads (new sets appear), and in snapshot isolation you can have write skew.
        - SI can have write skew because it only prevents write-write conflicts. It doesn't do any range checking or prohibit read-write conflicts (at least, vanilla SI doesn't). But if you do a `GET FOR UPDATE` then I suppose you get a bit stronger isolation.
        - The paper introduced SI and said, hey, this doesn't fit in your 4-level framework!
        - Repeatable read can have phantoms because it doesn't lock the entire range set of the table being selected. It only locks individual rows. Tricky 2PC here.
        - Predicate locking is needed for ANSI serializable mode (i.e., range locks + locks on the result set of a filter).
        - 2PC locking, versus snapshot isolation + OCC/locking, are the main approaches to a DB.
    - C-Store
        - First prototype of a column-oriented database in 2005.
        - Tuple mover moves data from the WS (writeable store) to the RS (read-optimized store). This gives you fast OLTP insertions in addition to OLAP.
        - Overlapping projections of tables, unlike tables, secondary indices (i.e., non-clustered indices), and projections (materialized views).
        - Much faster, and thanks to compression, also smaller in disk usage. Saving space via efficient data encodings with strings and other numeric types.
        - Tradeoff between read performance and write performance via these phases.
        - Also, the same column could be sorted on multiple attributes in separate projections, that's kind of fun to think about. Floating sort-orders.
    - Shark (Spark SQL)
        - Join strategies implemented on top of Spark RDDs. Enables interactive performance unlike past systems because the data is held in RAM before future computation work.
        - Adaptive join optimizer: shuffle join vs map join.
        - "Partial DAG Execution" does a few runs, then re-optimizes the query based on a profile.
        - I guess that makes sense, joins and aggregates are kind of the key part of a MapReduce algorithm that SQL needs to translate to. The table itself is an RDD partitioned over multiple machines, and then joins operate between those partitions.
        - In-memory columnar storage and compression helps reduce I/O and CPU usage.
