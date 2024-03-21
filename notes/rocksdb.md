# RocksDB 学习笔记

## 基本功能
1. Column Family 列族  
    列族提供了一种从逻辑上给数据库分片的方法。RocksDB的每个键值对都与唯一一个列族结合。如果没有指定Column Family，键值对将会结合到“default” 列族。不同列祖，配置信息也可以不一样。
    在 Pika 的存储引擎 Floyd 中，使用了 9 个列族来存放不同数据结构的数据，使用方法如下：
    - 初始化时：使用容器保存需要创建的列族描述对象 `ColumnFamilyDescriptor` ，传递给 Open 函数。并使用 `vector<ColumnFamilyHandle*>` 接收结果，这个容器中保存的指针是后续对列族操作时，指定列族的信息，顺序与之前保存描述对象的容器中的顺序一致。  
    以 Pika 为例子：
    ```C++
    Status Redis::Open(const StorageOptions& storage_options, const std::string& db_path) {
      rocksdb::DBOptions db_ops(storage_options.options);
      db_ops.create_missing_column_families = true;

      // string column-family options
      rocksdb::ColumnFamilyOptions string_cf_ops(storage_options.options);
      // hash column-family options
      rocksdb::ColumnFamilyOptions hash_meta_cf_ops(storage_options.options);
      rocksdb::ColumnFamilyOptions hash_data_cf_ops(storage_options.options);
      // list column-family options
      rocksdb::ColumnFamilyOptions list_meta_cf_ops(storage_options.options);
      rocksdb::ColumnFamilyOptions list_data_cf_ops(storage_options.options);
      // set column-family options
      rocksdb::ColumnFamilyOptions set_meta_cf_ops(storage_options.options);
      rocksdb::ColumnFamilyOptions set_data_cf_ops(storage_options.options);
      // zset column-family options
      rocksdb::ColumnFamilyOptions zset_meta_cf_ops(storage_options.options);
      rocksdb::ColumnFamilyOptions zset_data_cf_ops(storage_options.options);
      rocksdb::ColumnFamilyOptions zset_score_cf_ops(storage_options.options);

      std::vector<rocksdb::ColumnFamilyDescriptor> column_families;
      column_families.emplace_back(rocksdb::kDefaultColumnFamilyName, string_cf_ops);
      // hash CF
      column_families.emplace_back("hash_meta_cf", hash_meta_cf_ops);
      column_families.emplace_back("hash_data_cf", hash_data_cf_ops);
      // set CF
      column_families.emplace_back("set_meta_cf", set_meta_cf_ops);
      column_families.emplace_back("set_data_cf", set_data_cf_ops);
      // list CF
      column_families.emplace_back("list_meta_cf", list_meta_cf_ops);
      column_families.emplace_back("list_data_cf", list_data_cf_ops);
      // zset CF
      column_families.emplace_back("zset_meta_cf", zset_meta_cf_ops);
      column_families.emplace_back("zset_data_cf", zset_data_cf_ops);
      column_families.emplace_back("zset_score_cf", zset_score_cf_ops);
      return rocksdb::DB::Open(db_ops, db_path, column_families, &handles_, &db_);
    }
    ```
    - 读写时：需要指定 `ColumnFamilyHandle` 才能对指定列族操作，否则会对默认列族操作，同一个batch中可以对不同的列族操作。
    ```C++
      Status s = db_->Get(read_options, handles_[kHashesMetaCF], base_meta_key.Encode(), &meta_value);
      Status s = db_->Put(write_options_, handles_[kHashesMetaCF], base_meta_key.Encode(), meta_value);

      rocksdb::WriteBatch batch;
      batch.Put(handles_[kHashesMetaCF], base_meta_key.Encode(), meta_value);
      batch.Put(handles_[kHashesDataCF], hashes_data_key.Encode(), internal_value.Encode());
      s = db_->Write(default_write_options_, &batch);
    ```
2. Property  
    RocksDB 中的 DB 类中定义了一个结构体 Properties，里面是许多 RocksDB 的属性。 这些属性会在Flush的时候与kv数据一同存到 sst 文件中。还支持用户添加自己的 property。
    在 braft 项目中，我们就采用了自定义 property 特性来存储 braft log index 和 rocksdb sequence number 之间的映射关系：
    ```C++
    class LogIndexAndSequenceCollector {
    private:
      mutable std::mutex mutex_;
      std::list<LogIndexAndSequencePair> list_;
      class PairAndIterator {
      public:
        PairAndIterator() {}
        PairAndIterator(LogIndexAndSequencePair pair, decltype(list_)::iterator iter) : pair_(pair), iter_(iter) {}
        inline int64_t GetAppliedLogIndex() const { return pair_.GetAppliedLogIndex(); }
        inline rocksdb::SequenceNumber GetSequenceNumber() const { return pair_.GetSequenceNumber(); }
        inline decltype(list_)::iterator GetIterator() const { return iter_; }

      private:
        LogIndexAndSequencePair pair_;
        decltype(list_)::iterator iter_;
      };
      std::list<PairAndIterator> skip_list_;
      int64_t step_length_mask_ = 0;
      int64_t skip_length_mask_ = 0;

    public:
      explicit LogIndexAndSequenceCollector(uint8_t step_length_bit = 0, uint8_t extra_skip_length_bit = 8) {
        step_length_mask_ = (1 << step_length_bit) - 1;
        skip_length_mask_ = (1 << step_length_bit + extra_skip_length_bit) - 1;
      }

      void Update(int64_t smallest_applied_log_index, rocksdb::SequenceNumber smallest_flush_seqno);
      int64_t FindAppliedLogIndex(rocksdb::SequenceNumber seqno) const;

      template <typename T>
      void Purge(std::list<T> list, int64_t smallest_applied_log_index, int64_t smallest_flushed_log_index) {
        // purge condition:
        // We found first pair is greater than both smallest_flushed_log_index and smallest_applied_log_index,
        // then we keep previous one, and purge everyone before previous one.
        while (list.size() >= 2) {
          auto cur = list.begin();
          auto next = std::next(cur);
          if (smallest_flushed_log_index > cur->GetAppliedLogIndex() &&
              smallest_applied_log_index > next->GetAppliedLogIndex()) {
            list.pop_front();
          } else {
            break;
          }
        }
      }

      // purge out dated log index after memtable flushed.
      void Purge(int64_t smallest_applied_log_index, int64_t smallest_flushed_log_index) {
        std::lock_guard<std::mutex> guard(mutex_);
        Purge(list_, smallest_applied_log_index, smallest_flushed_log_index);
        Purge(skip_list_, smallest_applied_log_index, smallest_flushed_log_index);
      }
    };

    class LogIndexTablePropertiesCollector : public rocksdb::TablePropertiesCollector {
    public:
      explicit LogIndexTablePropertiesCollector(const LogIndexAndSequenceCollector *collector) : collector_(collector) {}

      rocksdb::Status AddUserKey(const rocksdb::Slice &key, const rocksdb::Slice &value, rocksdb::EntryType type,
                                rocksdb::SequenceNumber seq, uint64_t file_size) override;
      rocksdb::Status Finish(rocksdb::UserCollectedProperties *properties) override;
      const char *Name() const override { return "LogIndexTablePropertiesCollector"; }
      rocksdb::UserCollectedProperties GetReadableProperties() const override;

      static std::optional<LogIndexAndSequencePair> ReadStatsFromTableProps(
          const std::shared_ptr<const rocksdb::TableProperties> &table_props);
      static const inline std::string kPropertyName_{"latest-applied-log-index/largest-sequence-number"};

    private:
      std::pair<std::string, std::string> Materialize() const;

    private:
      const LogIndexAndSequenceCollector *collector_;
      rocksdb::SequenceNumber smallest_seqno_ = 0;
      rocksdb::SequenceNumber largest_seqno_ = 0;
      mutable std::map<rocksdb::SequenceNumber, int64_t> tmp_;
    };

    class LogIndexTablePropertiesCollectorFactory : public rocksdb::TablePropertiesCollectorFactory {
    public:
      explicit LogIndexTablePropertiesCollectorFactory(const LogIndexAndSequenceCollector *collector)
          : collector_(collector) {}
      ~LogIndexTablePropertiesCollectorFactory() override = default;

      rocksdb::TablePropertiesCollector *CreateTablePropertiesCollector(
          [[maybe_unused]] rocksdb::TablePropertiesCollectorFactory::Context context) override {
        return new LogIndexTablePropertiesCollector(collector_);
      }

      const char *Name() const override { return "LogIndexTablePropertiesCollectorFactory"; }

    private:
      const LogIndexAndSequenceCollector *collector_;
    };

    class LogIndexAndSequenceCollectorPurger : public rocksdb::EventListener {
    public:
      explicit LogIndexAndSequenceCollectorPurger(LogIndexAndSequenceCollector *collector, LogIndexOfCF *cf)
          : collector_(collector), cf_(cf) {}
      void OnFlushCompleted(rocksdb::DB *db, const rocksdb::FlushJobInfo &flush_job_info) override;

    private:
      LogIndexAndSequenceCollector *collector_;
      LogIndexOfCF *cf_;
    };
    ```

## 设计原理