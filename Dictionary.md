### 字典

#### 数据结构
1. entries：数据数组
2. buckets：哈希桶
key->hash_code->bucket_index->entry_index

#### 哈希冲突
1. 原因：当多个数据映射到同一个哈希桶时，就发生了哈希冲突。
2. 解决：拉链法——用单链表头插法将产生哈希冲突的数据串联起来。

#### 扩容机制
用比2倍原容量大的最小素数作为新容量进行扩容

Dictionary.cs
```
public class Dictionary<TKey, TValue>
{
    private struct Entry
    {
        public int    hashCode;
        public int    next;
        public TKey   key;
        public TValue value;
    }

    private int[]     buckets;
    private Entry[]   entries;
    private int       count;
    private int       freeList;
    private int       freeCount;

    public Dictionary() : this(0)
    { }

    public Dictionary(int capacity)
    {
        Initialize(capacity);
    }

    public int Count
    {
        get
        {
            return count - freeCount;
        }
    }

    public TValue this[TKey key]
    {
        get
        {
            int i = FindEntry(key);

            if (i >= 0)
                return entries[i].value;

            return default(TValue);
        }

        set
        {
            Insert(key, value);
        }
    }

    public void Add(TKey key, TValue value)
    {
        Insert(key, value);
    }

    public bool Remove(TKey key)
    {
        int hashCode = key.GetHashCode() & 0x7FFFFFFF;
        int bucket = hashCode % buckets.Length;
        int last = -1;
        for (int i = buckets[bucket]; i >= 0; last = i, i = entries[i].next)
        {
            if (entries[i].hashCode == hashCode && entries[i].key.Equals(key))
            {
                if (last < 0)
                    buckets[bucket] = entries[i].next;
                else
                    entries[last].next = entries[i].next;

                entries[i].hashCode = -1;
                entries[i].next = freeList;
                entries[i].key = default(TKey);
                entries[i].value = default(TValue);
                freeList = i;
                freeCount++;
                return true;
            }
        }

        return false;
    }

    public bool TryGetValue(TKey key, out TValue value)
    {
        int i = FindEntry(key);
        if (i >= 0)
        {
            value = entries[i].value;
            return true;
        }

        value = default(TValue);
        return false;
    }

    public bool ContainsKey(TKey key)
    {
        return FindEntry(key) >= 0;
    }

    public bool ContainsValue(TValue value)
    {
        if (value == null)
        {
            for (int i = 0; i < count; i++)
            {
                if (entries[i].hashCode >= 0 && entries[i].value == null)
                    return true;
            }
        }
        else
        {
            for (int i = 0; i < count; i++)
            {
                if (entries[i].hashCode >= 0 && entries[i].value.Equals(value))
                    return true;
            }
        }

        return false;
    }

    public void Clear()
    {
        if (count > 0)
        {
            for (int i = 0; i < buckets.Length; i++)
                buckets[i] = -1;

            System.Array.Clear(entries, 0, count);
            freeList = -1;
            count = 0;
            freeCount = 0;
        }
    }

    private void Initialize(int capacity)
    {
        int size = HashHelpers.GetPrime(capacity);
        buckets = new int[size];
        for (int i = 0; i < buckets.Length; i++)
            buckets[i] = -1;

        entries = new Entry[size];
        freeList = -1;
    }

    private int FindEntry(TKey key)
    {
        if (buckets != null)
        {

            int hashCode = key.GetHashCode() & 0x7FFFFFFF;
            for (int i = buckets[hashCode % buckets.Length]; i >= 0; i = entries[i].next)
            {
                if (entries[i].hashCode == hashCode && entries[i].key.Equals(key))
                    return i;
            }
        }

        return -1;
    }

    private void Insert(TKey key, TValue value)
    {
        int hashCode = key.GetHashCode() & 0x7FFFFFFF;
        int targetBucket = hashCode % buckets.Length;

        for (int i = buckets[targetBucket]; i >= 0; i = entries[i].next)
        {
            if (entries[i].hashCode == hashCode && entries[i].key.Equals(key))
            {
                entries[i].value = value;
                return;
            }
        }

        int index;
        if (freeCount > 0)
        {
            index = freeList;
            freeList = entries[index].next;
            freeCount--;
        }
        else
        {
            if (count == entries.Length)
            {
                Resize();
                targetBucket = hashCode % buckets.Length;
            }

            index = count;
            count++;
        }

        entries[index].hashCode = hashCode;
        entries[index].next = buckets[targetBucket];
        entries[index].key = key;
        entries[index].value = value;
        buckets[targetBucket] = index;
    }

    private void Resize()
    {
        Resize(HashHelpers.ExpandPrime(count));
    }

    private void Resize(int newSize)
    {
        int[] newBuckets = new int[newSize];
        for (int i = 0; i < newBuckets.Length; i++)
            newBuckets[i] = -1;

        Entry[] newEntries = new Entry[newSize];
        System.Array.Copy(entries, 0, newEntries, 0, count);

        for (int i = 0; i < count; i++)
        {
            if (newEntries[i].hashCode >= 0)
            {
                int bucket = newEntries[i].hashCode % newSize;
                newEntries[i].next = newBuckets[bucket];
                newBuckets[bucket] = i;
            }
        }

        buckets = newBuckets;
        entries = newEntries;
    }
}
```

HashHelpers.cs
```
internal static class HashHelpers
{
    public static readonly int[] primes = {
            3, 7, 11, 17, 23, 29, 37, 47, 59, 71, 89, 107, 131, 163, 197, 239, 293, 353, 431, 521, 631, 761, 919,
            1103, 1327, 1597, 1931, 2333, 2801, 3371, 4049, 4861, 5839, 7013, 8419, 10103, 12143, 14591,
            17519, 21023, 25229, 30293, 36353, 43627, 52361, 62851, 75431, 90523, 108631, 130363, 156437,
            187751, 225307, 270371, 324449, 389357, 467237, 560689, 672827, 807403, 968897, 1162687, 1395263,
            1674319, 2009191, 2411033, 2893249, 3471899, 4166287, 4999559, 5999471, 7199369};

    public static bool IsPrime(int candidate)
    {
        if ((candidate & 1) != 0)
        {
            int limit = (int)System.Math.Sqrt(candidate);
            for (int divisor = 3; divisor <= limit; divisor += 2)
            {
                if ((candidate % divisor) == 0)
                    return false;
            }

            return true;
        }

        return (candidate == 2);
    }

    public static int GetPrime(int min)
    {
        for (int i = 0; i < primes.Length; i++)
        {
            int prime = primes[i];
            if (prime >= min)
                return prime;
        }

        for (int i = (min | 1); i < System.Int32.MaxValue; i += 2)
        {
            if (IsPrime(i) && ((i - 1) % 101/*Hashtable.HashPrime*/ != 0))
                return i;
        }

        return min;
    }

    public static int GetMinPrime()
    {
        return primes[0];
    }

    public static int ExpandPrime(int oldSize)
    {
        int newSize = 2 * oldSize;

        if ((uint)newSize > MaxPrimeArrayLength && MaxPrimeArrayLength > oldSize)
            return MaxPrimeArrayLength;

        return GetPrime(newSize);
    }

    public const int MaxPrimeArrayLength = 0x7FEFFFFD;
}
```