common
===========

数据结构
------------------------

文件系统类型
^^^^^^^^^^^^

.. code-block:: c

  struct file_system_type {
    const char *name;
    int fs_flags;
  #define FS_REQUIRES_DEV		1
  #define FS_BINARY_MOUNTDATA	2
  #define FS_HAS_SUBTYPE		4
  #define FS_USERNS_MOUNT		8	/* Can be mounted by userns root */
  #define FS_DISALLOW_NOTIFY_PERM	16	/* Disable fanotify permission events */
  #define FS_THP_SUPPORT		8192	/* Remove once all fs converted */
  #define FS_RENAME_DOES_D_MOVE	32768	/* FS will handle d_move() during rename() internally. */
    int (*init_fs_context)(struct fs_context *);
    const struct fs_parameter_spec *parameters;
    struct dentry *(*mount) (struct file_system_type *, int,
            const char *, void *);
    void (*kill_sb) (struct super_block *);
    struct module *owner;
    struct file_system_type * next;
    struct hlist_head fs_supers;

    struct lock_class_key s_lock_key;
    struct lock_class_key s_umount_key;
    struct lock_class_key s_vfs_rename_key;
    struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];

    struct lock_class_key i_lock_key;
    struct lock_class_key i_mutex_key;
    struct lock_class_key i_mutex_dir_key;
  };

超级块
^^^^^^^

.. code-block:: c

  struct super_block {
    struct list_head	s_list;		/* Keep this first */
    dev_t			s_dev;		/* search index; _not_ kdev_t */
    unsigned char		s_blocksize_bits;
    unsigned long		s_blocksize;
    loff_t			s_maxbytes;	/* Max file size */
    struct file_system_type	*s_type;
    const struct super_operations	*s_op;
    const struct dquot_operations	*dq_op;
    const struct quotactl_ops	*s_qcop;
    const struct export_operations *s_export_op;
    unsigned long		s_flags;
    unsigned long		s_iflags;	/* internal SB_I_* flags */
    unsigned long		s_magic;
    struct dentry		*s_root;
    struct rw_semaphore	s_umount;
    int			s_count;
    atomic_t		s_active;
  #ifdef CONFIG_SECURITY
    void                    *s_security;
  #endif
    const struct xattr_handler **s_xattr;
  #ifdef CONFIG_FS_ENCRYPTION
    const struct fscrypt_operations	*s_cop;
    struct key		*s_master_keys; /* master crypto keys in use */
  #endif
  #ifdef CONFIG_FS_VERITY
    const struct fsverity_operations *s_vop;
  #endif
  #ifdef CONFIG_UNICODE
    struct unicode_map *s_encoding;
    __u16 s_encoding_flags;
  #endif
    struct hlist_bl_head	s_roots;	/* alternate root dentries for NFS */
    struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
    struct block_device	*s_bdev;
    struct backing_dev_info *s_bdi;
    struct mtd_info		*s_mtd;
    struct hlist_node	s_instances;
    unsigned int		s_quota_types;	/* Bitmask of supported quota types */
    struct quota_info	s_dquot;	/* Diskquota specific options */

    struct sb_writers	s_writers;

    /*
    * Keep s_fs_info, s_time_gran, s_fsnotify_mask, and
    * s_fsnotify_marks together for cache efficiency. They are frequently
    * accessed and rarely modified.
    */
    void			*s_fs_info;	/* Filesystem private info */

    /* Granularity of c/m/atime in ns (cannot be worse than a second) */
    u32			s_time_gran;
    /* Time limits for c/m/atime in seconds */
    time64_t		   s_time_min;
    time64_t		   s_time_max;
  #ifdef CONFIG_FSNOTIFY
    __u32			s_fsnotify_mask;
    struct fsnotify_mark_connector __rcu	*s_fsnotify_marks;
  #endif

    char			s_id[32];	/* Informational name */
    uuid_t			s_uuid;		/* UUID */

    unsigned int		s_max_links;
    fmode_t			s_mode;

    /*
    * The next field is for VFS *only*. No filesystems have any business
    * even looking at it. You had been warned.
    */
    struct mutex s_vfs_rename_mutex;	/* Kludge */

    /*
    * Filesystem subtype.  If non-empty the filesystem type field
    * in /proc/mounts will be "type.subtype"
    */
    const char *s_subtype;

    const struct dentry_operations *s_d_op; /* default d_op for dentries */

    /*
    * Saved pool identifier for cleancache (-1 means none)
    */
    int cleancache_poolid;

    struct shrinker s_shrink;	/* per-sb shrinker handle */

    /* Number of inodes with nlink == 0 but still referenced */
    atomic_long_t s_remove_count;

    /* Pending fsnotify inode refs */
    atomic_long_t s_fsnotify_inode_refs;

    /* Being remounted read-only */
    int s_readonly_remount;

    /* per-sb errseq_t for reporting writeback errors via syncfs */
    errseq_t s_wb_err;

    /* AIO completions deferred from interrupt context */
    struct workqueue_struct *s_dio_done_wq;
    struct hlist_head s_pins;

    /*
    * Owning user namespace and default context in which to
    * interpret filesystem uids, gids, quotas, device nodes,
    * xattrs and security labels.
    */
    struct user_namespace *s_user_ns;

    /*
    * The list_lru structure is essentially just a pointer to a table
    * of per-node lru lists, each of which has its own spinlock.
    * There is no need to put them into separate cachelines.
    */
    struct list_lru		s_dentry_lru;
    struct list_lru		s_inode_lru;
    struct rcu_head		rcu;
    struct work_struct	destroy_work;

    struct mutex		s_sync_lock;	/* sync serialisation lock */

    /*
    * Indicates how deep in a filesystem stack this SB is
    */
    int s_stack_depth;

    /* s_inode_list_lock protects s_inodes */
    spinlock_t		s_inode_list_lock ____cacheline_aligned_in_smp;
    struct list_head	s_inodes;	/* all inodes */

    spinlock_t		s_inode_wblist_lock;
    struct list_head	s_inodes_wb;	/* writeback inodes */
  } __randomize_layout;

目录项相关
^^^^^^^^^^^

.. code-block:: c

  struct dentry {
    /* RCU lookup touched fields */
    unsigned int d_flags;		/* protected by d_lock */
    seqcount_spinlock_t d_seq;	/* per dentry seqlock */
    struct hlist_bl_node d_hash;	/* lookup hash list */
    struct dentry *d_parent;	/* parent directory */
    struct qstr d_name;
    struct inode *d_inode;		/* Where the name belongs to - NULL is
            * negative */
    unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

    /* Ref lookup also touches following */
    struct lockref d_lockref;	/* per-dentry lock and refcount */
    const struct dentry_operations *d_op;
    struct super_block *d_sb;	/* The root of the dentry tree */
    unsigned long d_time;		/* used by d_revalidate */
    void *d_fsdata;			/* fs-specific data */

    union {
      struct list_head d_lru;		/* LRU list */
      wait_queue_head_t *d_wait;	/* in-lookup ones only */
    };
    struct list_head d_child;	/* child of parent list */
    struct list_head d_subdirs;	/* our children */
    /*
    * d_alias and d_rcu can share memory
    */
    union {
      struct hlist_node d_alias;	/* inode alias list */
      struct hlist_bl_node d_in_lookup_hash;	/* only for in-lookup ones */
      struct rcu_head d_rcu;
    } d_u;
  } __randomize_layout;

.. code-block:: c

  struct dentry_operations {
    int (*d_revalidate)(struct dentry *, unsigned int);
    int (*d_weak_revalidate)(struct dentry *, unsigned int);
    int (*d_hash)(const struct dentry *, struct qstr *);
    int (*d_compare)(const struct dentry *,
        unsigned int, const char *, const struct qstr *);
    int (*d_delete)(const struct dentry *);
    int (*d_init)(struct dentry *);
    void (*d_release)(struct dentry *);
    void (*d_prune)(struct dentry *);
    void (*d_iput)(struct dentry *, struct inode *);
    char *(*d_dname)(struct dentry *, char *, int);
    struct vfsmount *(*d_automount)(struct path *);
    int (*d_manage)(const struct path *, bool);
    struct dentry *(*d_real)(struct dentry *, const struct inode *);
  } ____cacheline_aligned;

inode
^^^^^^^^

.. code-block:: c

  /*
  * Keep mostly read-only and often accessed (especially for
  * the RCU path lookup and 'stat' data) fields at the beginning
  * of the 'struct inode'
  */
  struct inode {
    umode_t			i_mode;
    unsigned short		i_opflags;
    kuid_t			i_uid;
    kgid_t			i_gid;
    unsigned int		i_flags;

  #ifdef CONFIG_FS_POSIX_ACL
    struct posix_acl	*i_acl;
    struct posix_acl	*i_default_acl;
  #endif

    const struct inode_operations	*i_op;
    struct super_block	*i_sb;
    struct address_space	*i_mapping;

  #ifdef CONFIG_SECURITY
    void			*i_security;
  #endif

    /* Stat data, not accessed from path walking */
    unsigned long		i_ino;
    /*
    * Filesystems may only read i_nlink directly.  They shall use the
    * following functions for modification:
    *
    *    (set|clear|inc|drop)_nlink
    *    inode_(inc|dec)_link_count
    */
    union {
      const unsigned int i_nlink;
      unsigned int __i_nlink;
    };
    dev_t			i_rdev;
    loff_t			i_size;
    struct timespec64	i_atime;
    struct timespec64	i_mtime;
    struct timespec64	i_ctime;
    spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
    unsigned short          i_bytes;
    u8			i_blkbits;
    u8			i_write_hint;
    blkcnt_t		i_blocks;

  #ifdef __NEED_I_SIZE_ORDERED
    seqcount_t		i_size_seqcount;
  #endif

    /* Misc */
    unsigned long		i_state;
    struct rw_semaphore	i_rwsem;

    unsigned long		dirtied_when;	/* jiffies of first dirtying */
    unsigned long		dirtied_time_when;

    struct hlist_node	i_hash;
    struct list_head	i_io_list;	/* backing dev IO list */
  #ifdef CONFIG_CGROUP_WRITEBACK
    struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

    /* foreign inode detection, see wbc_detach_inode() */
    int			i_wb_frn_winner;
    u16			i_wb_frn_avg_time;
    u16			i_wb_frn_history;
  #endif
    struct list_head	i_lru;		/* inode LRU list */
    struct list_head	i_sb_list;
    struct list_head	i_wb_list;	/* backing dev writeback list */
    union {
      struct hlist_head	i_dentry;
      struct rcu_head		i_rcu;
    };
    atomic64_t		i_version;
    atomic64_t		i_sequence; /* see futex */
    atomic_t		i_count;
    atomic_t		i_dio_count;
    atomic_t		i_writecount;
  #if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
    atomic_t		i_readcount; /* struct files open RO */
  #endif
    union {
      const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
      void (*free_inode)(struct inode *);
    };
    struct file_lock_context	*i_flctx;
    struct address_space	i_data;
    struct list_head	i_devices;
    union {
      struct pipe_inode_info	*i_pipe;
      struct block_device	*i_bdev;
      struct cdev		*i_cdev;
      char			*i_link;
      unsigned		i_dir_seq;
    };

    __u32			i_generation;

  #ifdef CONFIG_FSNOTIFY
    __u32			i_fsnotify_mask; /* all events this inode cares about */
    struct fsnotify_mark_connector __rcu	*i_fsnotify_marks;
  #endif

  #ifdef CONFIG_FS_ENCRYPTION
    struct fscrypt_info	*i_crypt_info;
  #endif

  #ifdef CONFIG_FS_VERITY
    struct fsverity_info	*i_verity_info;
  #endif

    void			*i_private; /* fs or device private pointer */
  } __randomize_layout;

接口
------------------------