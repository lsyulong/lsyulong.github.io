---
layout: mypost
title: Swanlake-Improve prepared-statement execution error and param tests
categories: [swanlake]
---

## 问题描述  
- **Issue**:#115 - Improve prepared-statement execution error and param tests    
- **标签**: testing, prepared-statements  
- **PR**: https://github.com/swanlake-io/swanlake/pull/135

## 技术分析  
### 涉及模块  
- `swanlake-core/src/service/execute.rs` - 预处理语句执行逻辑 
- `tests/runner/src/scenarios/prepared_statements.rs` - 测试场景 
  
### 核心问题  
1. 参数数量不匹配时错误处理不够友好  
2. NULL 参数处理缺乏回归测试  
3. 临时预处理语句行为未验证

## 解决方案  
### 实现的测试用例  
  
#### 1. 参数不匹配测试  
```rust  
    fn test_parameter_mismatch_errors(&mut self) -> Result<()> {
        self.drop_table_if_exists("param_mismatch_test")?;
        let test_result = (|| -> Result<()> {
            //create test table
            self.update("CREATE TABLE param_mismatch_test(col1 INTEGER,col2 INTEGER)")?;

            //case1 too few parameters
            let single_param_schema = Arc::new(Schema::new(vec![Field::new(
                "col1",
                DataType::Int32,
                false,
            )]));
            let single_param_batch = RecordBatch::try_new(
                single_param_schema,
                vec![Arc::new(Int32Array::from(vec![1])) as ArrayRef],
            )?;
            let result = self.client.update_with_record_batch(
                "INSERT INTO param_mismatch_test (col1, col2) VALUES (?, ?)",
                single_param_batch,
            );
            assert!(result.is_err());

            //case2 too many parameters
            let double_param_schema = Arc::new(Schema::new(vec![
                Field::new("col1", DataType::Int32, false),
                Field::new("col2", DataType::Int32, false),
            ]));
            let double_param_batch = RecordBatch::try_new(
                double_param_schema,
                vec![
                    Arc::new(Int32Array::from(vec![1])) as ArrayRef,
                    Arc::new(Int32Array::from(vec![1])) as ArrayRef,
                ],
            )?;

            let result = self.client.update_with_record_batch(
                "INSERT INTO param_mismatch_test (col1) VALUES (?)",
                double_param_batch,
            );
            assert!(result.is_err());
            Ok(())
        })();

        self.finalize_table("param_mismatch_test", test_result)
    }
```

#### 2. NULL 参数测试
```rust 
    fn test_null_empty_parameters(&mut self) -> Result<()> {
        self.drop_table_if_exists("null_empty_test")?;
        let test_result = (|| -> Result<()> {
            //create table
            self.update("CREATE TABLE null_empty_test (id INTEGER, name VARCHAR, value INTEGER)")?;

            //test null parameter
            let null_schema = Arc::new(Schema::new(vec![
                Field::new("id", DataType::Int32, false),
                Field::new("name", DataType::Utf8, true),
                Field::new("value", DataType::Int32, true),
            ]));

            let null_batch = RecordBatch::try_new(
                null_schema,
                vec![
                    Arc::new(Int32Array::from(vec![1])) as ArrayRef,
                    Arc::new(StringArray::from(vec![Option::<&str>::None])) as ArrayRef,
                    Arc::new(Int32Array::from(vec![Option::<i32>::None])) as ArrayRef,
                ],
            )?;

            self.client.update_with_record_batch(
                "INSERT INTO null_empty_test (id, name, value) VALUES (?, ?, ?)",
                null_batch,
            )?;

            //verify null values were inserted
            let result = self
                .client
                .query("SELECT name, value FROM null_empty_test WHERE id = 1")?;
            let batch = result
                .batches
                .first()
                .context("expected row after null insert")?;
            assert!(batch.column(0).is_null(0));
            assert!(batch.column(1).is_null(0));
            Ok(())
        })();

        self.finalize_table("null_empty_test", test_result)
    }
```

#### 3. 临时语句测试
```rust 
    fn test_ephemeral_prepared_statements(&mut self) -> Result<()> {
        self.drop_table_if_exists("ephemeral_test")?;
        let test_result = (|| -> Result<()> {
            //create table
            self.update("CREATE TABLE ephemeral_test (id INTEGER, name VARCHAR)")?;
            self.update("INSERT INTO ephemeral_test VALUES (1, 'test')")?;

            //test with regular prepared statement(for comparison)
            let regular_params = RecordBatch::try_new(
                Arc::new(Schema::new(vec![Field::new("id", DataType::Int32, false)])),
                vec![Arc::new(Int32Array::from(vec![1])) as ArrayRef],
            )?;

            let regular_result = self.client.query_with_param(
                "SELECT name FROM ephemeral_test WHERE id = ?",
                regular_params,
            )?;
            assert_eq!(regular_result.total_rows, 1);

            Ok(())
        })();

        self.finalize_table("ephemeral_test", test_result)
    }
}
```

## 学习心得  
### 技术收获  
1. **Rust 测试模式**: 学习了 SwanLake 的表创建-测试-清理模式  
2. **Arrow 数据类型**: 掌握了 `RecordBatch` 和 `ArrayRef` 的使用  

  
### 遇到的挑战  
1. **模式匹配错误**: 需要导入 `Result::Ok` 变体  
2. **字段名问题**: 必须使用字符串字面量  

  
### 解决过程  
记录编译错误和调试过程，展示问题解决思路。
笔者当前环境centos7
编译报错，很明显系统自带的openssl和项目依赖的openssl不兼容
```bash 
[root@poc-05-44 dev]# cd /data/soft/jdk/swanlake/
[root@poc-05-44 swanlake]# git pull https://github.com/lsyulong/swanlake.git
remote: Enumerating objects: 21, done.
remote: Counting objects: 100% (16/16), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 9 (delta 4), reused 7 (delta 4), pack-reused 0 (from 0)
Unpacking objects: 100% (9/9), done.
From https://github.com/lsyulong/swanlake
 * branch            HEAD       -> FETCH_HEAD
Updating b17438c..700bc0c
Fast-forward
 tests/runner/src/scenarios/prepared_statements.rs | 85 ++++++++++++++++++++-
 1 file changed, 84 insertions(+), 1 deletion(-)
[root@poc-05-44 swanlake]# cargo build 
   Compiling unicode-ident v1.0.24
   Compiling cfg-if v1.0.4
   Compiling bytes v1.11.1
   Compiling once_cell v1.21.4
   Compiling memchr v2.8.0
   Compiling libc v0.2.185
   Compiling pin-project-lite v0.2.17
   Compiling serde_core v1.0.228
   Compiling proc-macro2 v1.0.106
   Compiling itoa v1.0.18
   Compiling futures-core v0.3.32
   Compiling futures-sink v0.3.32
   Compiling jobserver v0.1.34
   Compiling mio v1.2.0
   Compiling socket2 v0.6.3
   Compiling smallvec v1.15.1
   Compiling libm v0.2.16
   Compiling cc v1.2.60
   Compiling slab v0.4.12
   Compiling futures-task v0.3.32
   Compiling quote v1.0.45
   Compiling futures-io v0.3.32
   Compiling bitflags v2.11.1
   Compiling futures-channel v0.3.32
   Compiling hashbrown v0.17.0
   Compiling equivalent v1.0.2
   Compiling tower-service v0.3.3
   Compiling try-lock v0.2.5
   Compiling syn v2.0.117
   Compiling want v0.3.1
   Compiling httparse v1.10.1
   Compiling percent-encoding v2.3.2
   Compiling ryu v1.0.23
   Compiling tokio v1.52.1
   Compiling iana-time-zone v0.1.65
   Compiling zmij v1.0.21
   Compiling http v1.4.0
   Compiling base64 v0.22.1
   Compiling fnv v1.0.7
   Compiling log v0.4.29
   Compiling indexmap v2.14.0
   Compiling openssl-sys v0.9.113
   Compiling getrandom v0.3.4
   Compiling form_urlencoded v1.2.2
   Compiling arrow-schema v58.1.0
   Compiling num-traits v0.2.19
   Compiling http-body v1.0.1
   Compiling futures-util v0.3.32
   Compiling num-integer v0.1.46
   Compiling chrono v0.4.44
   Compiling num-complex v0.4.6
   Compiling num-bigint v0.4.6
   Compiling hashbrown v0.16.1
   Compiling errno v0.3.14
   Compiling tracing-core v0.1.36
   Compiling atomic-waker v1.1.2
   Compiling tracing v0.1.44
   Compiling signal-hook-registry v1.4.8
   Compiling synstructure v0.13.2
   Compiling httpdate v1.0.3
   Compiling serde v1.0.228
error: failed to run custom build command for `openssl-sys v0.9.113`

Caused by:
  process didn't exit successfully: `/data/soft/jdk/swanlake/target/debug/build/openssl-sys-8813828a7e6131dd/build-script-main` (exit status: 101)
  --- stdout
  cargo:rustc-check-cfg=cfg(osslconf, values("OPENSSL_NO_OCB", "OPENSSL_NO_SM4", "OPENSSL_NO_SEED", "OPENSSL_NO_CHACHA", "OPENSSL_NO_CAST", "OPENSSL_NO_IDEA", "OPENSSL_NO_CAMELLIA", "OPENSSL_NO_RC4", "OPENSSL_NO_BF", "OPENSSL_NO_PSK", "OPENSSL_NO_DEPRECATED_3_0", "OPENSSL_NO_SCRYPT", "OPENSSL_NO_SM3", "OPENSSL_NO_RMD160", "OPENSSL_NO_EC2M", "OPENSSL_NO_OCSP", "OPENSSL_NO_CMS", "OPENSSL_NO_COMP", "OPENSSL_NO_SOCK", "OPENSSL_NO_STDIO", "OPENSSL_NO_EC", "OPENSSL_NO_SSL3_METHOD", "OPENSSL_NO_KRB5", "OPENSSL_NO_TLSEXT", "OPENSSL_NO_SRP", "OPENSSL_NO_SRTP", "OPENSSL_NO_RFC3779", "OPENSSL_NO_SHA", "OPENSSL_NO_NEXTPROTONEG", "OPENSSL_NO_ENGINE", "OPENSSL_NO_BUF_FREELISTS", "OPENSSL_NO_RC2"))
  cargo:rustc-check-cfg=cfg(openssl)
  cargo:rustc-check-cfg=cfg(libressl)
  cargo:rustc-check-cfg=cfg(boringssl)
  cargo:rustc-check-cfg=cfg(awslc)
  cargo:rustc-check-cfg=cfg(awslc_pregenerated)
  cargo:rustc-check-cfg=cfg(libressl250)
  cargo:rustc-check-cfg=cfg(libressl251)
  cargo:rustc-check-cfg=cfg(libressl252)
  cargo:rustc-check-cfg=cfg(libressl261)
  cargo:rustc-check-cfg=cfg(libressl270)
  cargo:rustc-check-cfg=cfg(libressl271)
  cargo:rustc-check-cfg=cfg(libressl273)
  cargo:rustc-check-cfg=cfg(libressl280)
  cargo:rustc-check-cfg=cfg(libressl281)
  cargo:rustc-check-cfg=cfg(libressl291)
  cargo:rustc-check-cfg=cfg(libressl310)
  cargo:rustc-check-cfg=cfg(libressl321)
  cargo:rustc-check-cfg=cfg(libressl332)
  cargo:rustc-check-cfg=cfg(libressl340)
  cargo:rustc-check-cfg=cfg(libressl350)
  cargo:rustc-check-cfg=cfg(libressl360)
  cargo:rustc-check-cfg=cfg(libressl361)
  cargo:rustc-check-cfg=cfg(libressl370)
  cargo:rustc-check-cfg=cfg(libressl380)
  cargo:rustc-check-cfg=cfg(libressl381)
  cargo:rustc-check-cfg=cfg(libressl382)
  cargo:rustc-check-cfg=cfg(libressl390)
  cargo:rustc-check-cfg=cfg(libressl400)
  cargo:rustc-check-cfg=cfg(libressl410)
  cargo:rustc-check-cfg=cfg(libressl420)
  cargo:rustc-check-cfg=cfg(libressl430)
  cargo:rustc-check-cfg=cfg(ossl101)
  cargo:rustc-check-cfg=cfg(ossl102)
  cargo:rustc-check-cfg=cfg(ossl102f)
  cargo:rustc-check-cfg=cfg(ossl102h)
  cargo:rustc-check-cfg=cfg(ossl110)
  cargo:rustc-check-cfg=cfg(ossl110f)
  cargo:rustc-check-cfg=cfg(ossl110g)
  cargo:rustc-check-cfg=cfg(ossl110h)
  cargo:rustc-check-cfg=cfg(ossl111)
  cargo:rustc-check-cfg=cfg(ossl111b)
  cargo:rustc-check-cfg=cfg(ossl111c)
  cargo:rustc-check-cfg=cfg(ossl111d)
  cargo:rustc-check-cfg=cfg(ossl300)
  cargo:rustc-check-cfg=cfg(ossl310)
  cargo:rustc-check-cfg=cfg(ossl320)
  cargo:rustc-check-cfg=cfg(ossl330)
  cargo:rustc-check-cfg=cfg(ossl340)
  cargo:rerun-if-env-changed=X86_64_UNKNOWN_LINUX_GNU_OPENSSL_LIB_DIR
  X86_64_UNKNOWN_LINUX_GNU_OPENSSL_LIB_DIR unset
  cargo:rerun-if-env-changed=OPENSSL_LIB_DIR
  OPENSSL_LIB_DIR unset
  cargo:rerun-if-env-changed=X86_64_UNKNOWN_LINUX_GNU_OPENSSL_INCLUDE_DIR
  X86_64_UNKNOWN_LINUX_GNU_OPENSSL_INCLUDE_DIR unset
  cargo:rerun-if-env-changed=OPENSSL_INCLUDE_DIR
  OPENSSL_INCLUDE_DIR unset
  cargo:rerun-if-env-changed=X86_64_UNKNOWN_LINUX_GNU_OPENSSL_DIR
  X86_64_UNKNOWN_LINUX_GNU_OPENSSL_DIR unset
  cargo:rerun-if-env-changed=OPENSSL_DIR
  OPENSSL_DIR unset
  cargo:rerun-if-env-changed=OPENSSL_NO_PKG_CONFIG
  cargo:rerun-if-env-changed=PKG_CONFIG_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG
  cargo:rerun-if-env-changed=PKG_CONFIG
  cargo:rerun-if-env-changed=OPENSSL_STATIC
  cargo:rerun-if-env-changed=OPENSSL_DYNAMIC
  cargo:rerun-if-env-changed=PKG_CONFIG_ALL_STATIC
  cargo:rerun-if-env-changed=PKG_CONFIG_ALL_DYNAMIC
  cargo:rerun-if-env-changed=PKG_CONFIG_PATH_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_PATH_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG_PATH
  cargo:rerun-if-env-changed=PKG_CONFIG_PATH
  cargo:rerun-if-env-changed=PKG_CONFIG_LIBDIR_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_LIBDIR_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG_LIBDIR
  cargo:rerun-if-env-changed=PKG_CONFIG_LIBDIR
  cargo:rerun-if-env-changed=PKG_CONFIG_SYSROOT_DIR_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_SYSROOT_DIR_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG_SYSROOT_DIR
  cargo:rerun-if-env-changed=PKG_CONFIG_SYSROOT_DIR
  cargo:rerun-if-env-changed=PKG_CONFIG_SYSROOT_DIR
  cargo:rerun-if-env-changed=SYSROOT
  cargo:rerun-if-env-changed=OPENSSL_STATIC
  cargo:rerun-if-env-changed=OPENSSL_DYNAMIC
  cargo:rerun-if-env-changed=PKG_CONFIG_ALL_STATIC
  cargo:rerun-if-env-changed=PKG_CONFIG_ALL_DYNAMIC
  cargo:rustc-link-lib=ssl
  cargo:rustc-link-lib=crypto
  cargo:rerun-if-env-changed=PKG_CONFIG_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG
  cargo:rerun-if-env-changed=PKG_CONFIG
  cargo:rerun-if-env-changed=OPENSSL_STATIC
  cargo:rerun-if-env-changed=OPENSSL_DYNAMIC
  cargo:rerun-if-env-changed=PKG_CONFIG_ALL_STATIC
  cargo:rerun-if-env-changed=PKG_CONFIG_ALL_DYNAMIC
  cargo:rerun-if-env-changed=PKG_CONFIG_PATH_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_PATH_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG_PATH
  cargo:rerun-if-env-changed=PKG_CONFIG_PATH
  cargo:rerun-if-env-changed=PKG_CONFIG_LIBDIR_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_LIBDIR_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG_LIBDIR
  cargo:rerun-if-env-changed=PKG_CONFIG_LIBDIR
  cargo:rerun-if-env-changed=PKG_CONFIG_SYSROOT_DIR_x86_64-unknown-linux-gnu
  cargo:rerun-if-env-changed=PKG_CONFIG_SYSROOT_DIR_x86_64_unknown_linux_gnu
  cargo:rerun-if-env-changed=HOST_PKG_CONFIG_SYSROOT_DIR
  cargo:rerun-if-env-changed=PKG_CONFIG_SYSROOT_DIR
  cargo:rerun-if-changed=build/expando.c
  cargo:rerun-if-env-changed=CC_x86_64-unknown-linux-gnu
  CC_x86_64-unknown-linux-gnu = None
  cargo:rerun-if-env-changed=CC_x86_64_unknown_linux_gnu
  CC_x86_64_unknown_linux_gnu = None
  cargo:rerun-if-env-changed=HOST_CC
  HOST_CC = None
  cargo:rerun-if-env-changed=CC
  CC = None
  cargo:rerun-if-env-changed=CC_ENABLE_DEBUG_OUTPUT
  cargo:rerun-if-env-changed=CRATE_CC_NO_DEFAULTS
  CRATE_CC_NO_DEFAULTS = None
  cargo:rerun-if-env-changed=CFLAGS
  CFLAGS = None
  cargo:rerun-if-env-changed=HOST_CFLAGS
  HOST_CFLAGS = None
  cargo:rerun-if-env-changed=CFLAGS_x86_64_unknown_linux_gnu
  CFLAGS_x86_64_unknown_linux_gnu = None
  cargo:rerun-if-env-changed=CFLAGS_x86_64-unknown-linux-gnu
  CFLAGS_x86_64-unknown-linux-gnu = None
  cargo:rustc-cfg=osslconf="OPENSSL_NO_EC2M"
  cargo:rustc-cfg=osslconf="OPENSSL_NO_SRP"
  cargo:conf=OPENSSL_NO_EC2M,OPENSSL_NO_SRP
  cargo:rustc-cfg=openssl
  cargo:rustc-cfg=ossl101
  cargo:rustc-cfg=ossl102
  cargo:rustc-cfg=ossl102f
  cargo:rustc-cfg=ossl102h
  cargo:rustc-cfg=ossl110
  cargo:version_number=100020bf

  --- stderr

  thread 'main' (25289) panicked at /root/.cargo/registry/src/rsproxy.cn-e3de039b2554c837/openssl-sys-0.9.113/build/main.rs:460:5:


  This crate is only compatible with OpenSSL (version 1.1.0, 1.1.1, or 3.x), or LibreSSL 3.5.0
  through 4.2.x, but a different version of OpenSSL was found. The build is now aborting
  due to this version mismatch.


  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
warning: build failed, waiting for other jobs to finish...
```

因为考虑到当前服务器不是测试机，所以不能直接进行强制升级openssl，可能会有其他问题出现(yum、ssh 无法使用)
临时解决方案， 安装 OpenSSL 1.1.1（并行安装，不替换系统默认）
```bash 
yum install -y openssl11 openssl11-libs openssl11-devel
```
安装完成后，编译项目时，指定使用openssl 1.1.1
```bash
# 临时指定 openssl 路径
export OPENSSL_LIB_DIR=/usr/lib64/openssl11
export OPENSSL_INCLUDE_DIR=/usr/include/openssl11
export LD_LIBRARY_PATH=/usr/lib64/openssl11:$LD_LIBRARY_PATH

# 清理并编译或者直接进行编译
cargo clean
cargo build --release
```
```bash 
[root@poc-05-44 swanlake]# ll /usr/lib64/openssl11
total 0
lrwxrwxrwx. 1 root root 22 Apr 28 17:10 libcrypto.so -> ../libcrypto.so.1.1.1k
lrwxrwxrwx. 1 root root 19 Apr 28 17:10 libssl.so -> ../libssl.so.1.1.1k
[root@poc-05-44 swanlake]# export OPENSSL_LIB_DIR=/usr/lib64/openssl11
[root@poc-05-44 swanlake]# export OPENSSL_INCLUDE_DIR=/usr/include/openssl11
[root@poc-05-44 swanlake]# export LD_LIBRARY_PATH=/usr/lib64/openssl11:$LD_LIBRARY_PATH
[root@poc-05-44 swanlake]# cargo build --release
   Compiling regex-automata v0.4.14
   Compiling arrow-select v58.1.0
   Compiling openssl-sys v0.9.113
   Compiling rustix v0.38.44
   Compiling ring v0.17.14
   Compiling serde_json v1.0.149
   Compiling http-body v1.0.1
   Compiling lexical-write-integer v1.0.6
   Compiling lexical-parse-integer v1.0.6
   Compiling http v1.4.0
   Compiling semver v1.0.28
   Compiling zeroize v1.8.2
   Compiling rustls-pki-types v1.14.0
   Compiling rustc_version v0.4.1
   Compiling lexical-write-float v1.0.6
   Compiling lexical-parse-float v1.0.6
   Compiling hybrid-array v0.4.10
   Compiling block-buffer v0.10.4
   Compiling httparse v1.10.1
   Compiling zstd-sys v2.0.16+zstd.1.5.7
   Compiling crossterm v0.28.1
   Compiling percent-encoding v2.3.2
   Compiling try-lock v0.2.5
   Compiling fnv v1.0.7
   Compiling foreign-types-shared v0.1.1
   Compiling atomic-waker v1.1.2
   Compiling tower-service v0.3.3
   Compiling untrusted v0.9.0
   Compiling unicode-width v0.2.2
   Compiling h2 v0.4.13
   Compiling foreign-types v0.3.2
   Compiling http v0.2.12
   Compiling lexical-core v1.0.6
   Compiling arrow-ord v58.1.0
   Compiling regex v1.12.3
   Compiling comfy-table v7.1.4
   Compiling want v0.3.1
   Compiling digest v0.10.7
   Compiling flatbuffers v25.12.19
   Compiling openssl v0.10.77
   Compiling atoi v2.0.0
   Compiling cpufeatures v0.2.17
   Compiling native-tls v0.2.18
   Compiling crunchy v0.2.4
   Compiling httpdate v1.0.3
   Compiling rustls v0.23.38
   Compiling cmov v0.5.3
   Compiling object v0.37.3
   Compiling hyper v1.9.0
   Compiling ctutils v0.4.2
   Compiling arrow-cast v58.1.0
   Compiling rustls-webpki v0.103.12
   Compiling crypto-common v0.2.1
   Compiling block-buffer v0.12.0
   Compiling anyhow v1.0.102
   Compiling tokio-util v0.7.18
   Compiling async-trait v0.1.89
   Compiling bzip2-sys v0.1.13+1.0.8
   Compiling openssl-probe v0.2.1
   Compiling sync_wrapper v1.0.2
   Compiling zstd-safe v5.0.2+zstd.1.5.2
   Compiling either v1.15.0
   Compiling rustix v1.1.4
   Compiling getrandom v0.4.2
   Compiling const-oid v0.10.2
   Compiling rand_core v0.10.1
   Compiling tiny-keccak v2.0.2
   Compiling tower-layer v0.3.3
   Compiling digest v0.11.2
   Compiling itertools v0.14.0
   Compiling h2 v0.3.27
   Compiling hyper-util v0.1.20
   Compiling http-body v0.4.6
   Compiling form_urlencoded v1.2.2
   Compiling http-body-util v0.1.3
   Compiling inout v0.1.4
   Compiling csv-core v0.1.13
   Compiling socket2 v0.5.10
   Compiling winnow v1.0.1
   Compiling powerfmt v0.2.0
   Compiling linux-raw-sys v0.12.1
   Compiling tinyvec_macros v0.1.1
   Compiling rand_core v0.6.4
   Compiling base64ct v1.8.3
   Compiling mime v0.3.17
   Compiling base64 v0.22.1
   Compiling cpufeatures v0.3.0
   Compiling axum-core v0.5.6
   Compiling ar_archive_writer v0.5.1
   Compiling tower v0.5.3
   Compiling tokio-rustls v0.26.4
   Compiling password-hash v0.4.2
   Compiling toml_parser v1.1.2+spec-1.1.0
   Compiling hyper v0.14.32
   Compiling tinyvec v1.11.0
   Compiling deranged v0.5.8
   Compiling csv v1.4.0
   Compiling cipher v0.4.4
   Compiling serde_urlencoded v0.7.1
   Compiling arrow-ipc v58.1.0
   Compiling prost-derive v0.14.3
   Compiling tokio-native-tls v0.3.1
   Compiling sha2 v0.10.9
   Compiling hmac v0.12.1
   Compiling arrow-string v58.1.0
   Compiling webpki-roots v1.0.7
   Compiling arrow-row v58.1.0
   Compiling arrow-arith v58.1.0
   Compiling serde_path_to_error v0.1.20
   Compiling pin-project-internal v1.1.11
   Compiling serde_spanned v1.1.1
   Compiling toml_datetime v1.1.1+spec-1.1.0
   Compiling time-core v0.1.8
   Compiling base64 v0.21.7
   Compiling toml_writer v1.1.1+spec-1.1.0
   Compiling typeid v1.0.3
   Compiling matchit v0.8.4
   Compiling bumpalo v3.20.2
   Compiling ucd-trie v0.1.7
   Compiling simdutf8 v0.1.5
   Compiling iri-string v0.7.12
   Compiling num-conv v0.2.1
   Compiling time v0.3.47
   Compiling arrow-json v58.1.0
   Compiling pest v2.8.6
   Compiling tower-http v0.6.8
   Compiling axum v0.8.9
   Compiling zopfli v0.8.3
   Compiling toml v1.1.2+spec-1.1.0
   Compiling rustls-pemfile v1.0.4
   Compiling pin-project v1.1.11
   Compiling pbkdf2 v0.11.0
   Compiling hyper-rustls v0.27.9
   Compiling prost v0.14.3
   Compiling hyper-tls v0.5.0
   Compiling zstd v0.11.2+zstd.1.5.2
   Compiling bzip2 v0.4.4
   Compiling const-random-macro v0.1.16
   Compiling arrow-csv v58.1.0
   Compiling xattr v1.6.1
   Compiling aes v0.8.4
   Compiling unicode-normalization v0.1.25
   Compiling psm v0.1.30
   Compiling chacha20 v0.10.0
   Compiling hyper-timeout v0.5.2
   Compiling sha1 v0.10.6
   Compiling tokio-stream v0.1.18
   Compiling filetime v0.2.27
   Compiling encoding_rs v0.8.35
   Compiling unicode-bidi v0.3.18
   Compiling foldhash v0.1.5
   Compiling sync_wrapper v0.1.2
   Compiling constant_time_eq v0.1.5
   Compiling unicode-properties v0.1.4
   Compiling byteorder v1.5.0
   Compiling reqwest v0.11.27
   Compiling zip v0.6.6
   Compiling stringprep v0.1.5
   Compiling hashbrown v0.15.5
   Compiling tonic v0.14.5
   Compiling reqwest v0.12.28
   Compiling rand v0.10.1
   Compiling tar v0.4.45
   Compiling arrow v58.1.0
   Compiling const-random v0.1.18
   Compiling zip v6.0.0
   Compiling pest_meta v2.8.6
   Compiling sha2 v0.11.0
   Compiling hmac v0.13.0
   Compiling md-5 v0.11.0
   Compiling stacker v0.1.23
   Compiling paste v1.0.15
   Compiling fallible-iterator v0.2.0
   Compiling erased-serde v0.4.10
   Compiling siphasher v1.0.2
   Compiling libduckdb-sys v1.10502.0
   Compiling phf_shared v0.13.1
   Compiling postgres-protocol v0.6.11
   Compiling pest_generator v2.8.6
   Compiling tonic-prost v0.14.5
   Compiling dlv-list v0.5.2
   Compiling adbc-driver-flightsql v0.1.2
   Compiling hashlink v0.10.0
   Compiling futures-executor v0.3.32
   Compiling adbc_core v0.23.0
   Compiling adbc_driver_manager v0.23.0
   Compiling heck v0.5.0
   Compiling lazy_static v1.5.0
   Compiling rust_decimal v1.41.0
   Compiling hashbrown v0.14.5
   Compiling sharded-slab v0.1.7
   Compiling ordered-multimap v0.7.3
   Compiling strum_macros v0.27.2
   Compiling adbc_ffi v0.23.0
   Compiling futures v0.3.32
   Compiling pest_derive v2.8.6
   Compiling postgres-types v0.2.13
   Compiling phf v0.13.1
   Compiling prost-types v0.14.3
   Compiling matchers v0.2.0
   Compiling tracing-serde v0.2.0
   Compiling tracing-log v0.2.0
   Compiling num-rational v0.4.2
   Compiling num-iter v0.1.45
   Compiling recursive-proc-macro-impl v0.1.1
   Compiling libloading v0.8.9
   Compiling whoami v2.1.1
   Compiling thread_local v1.1.9
   Compiling nu-ansi-term v0.50.3
   Compiling unicode-ident v1.0.24
   Compiling path-slash v0.2.1
   Compiling arrayvec v0.7.6
   Compiling arraydeque v0.5.1
   Compiling thiserror v2.0.18
   Compiling ron v0.12.1
   Compiling tracing-subscriber v0.3.23
   Compiling yaml-rust2 v0.10.4
   Compiling tokio-postgres v0.7.17
   Compiling recursive v0.1.1
   Compiling num v0.4.3
   Compiling arrow-flight v58.1.0
   Compiling strum v0.27.2
   Compiling serde-untagged v0.1.9
   Compiling json5 v0.4.1
   Compiling rust-ini v0.21.3
   Compiling convert_case v0.6.0
   Compiling thiserror-impl v2.0.18
   Compiling sqlparser_derive v0.5.0
   Compiling fallible-iterator v0.3.0
   Compiling cast v0.3.0
   Compiling pathdiff v0.2.3
   Compiling fallible-streaming-iterator v0.1.9
   Compiling config v0.15.22
   Compiling sqlparser v0.61.0
   Compiling postgres-native-tls v0.5.3
   Compiling swanlake-client v0.1.1 (/data/soft/jdk/swanlake/swanlake-client)
   Compiling duckdb v1.10502.0
   Compiling uuid v1.23.1
   Compiling tonic-health v0.14.5
   Compiling hex v0.4.3
   Compiling dotenvy v0.15.7
   Compiling rust-adbc v0.1.0 (/data/soft/jdk/swanlake/examples/rust-adbc)
   Compiling swanlake-runner v0.1.0 (/data/soft/jdk/swanlake/tests/runner)
error[E0532]: expected a pattern, found a function call
   --> tests/runner/src/scenarios/prepared_statements.rs:577:14
    |
577 |             (Ok(()), Ok(())) => Ok(()),
    |              ^^ not a tuple struct or tuple variant
    |
    = note: function calls are not allowed in patterns: <https://doc.rust-lang.org/book/ch19-00-patterns.html>
help: consider importing this tuple variant instead
    |
  1 + use std::result::Result::Ok;
    |

error[E0532]: expected a pattern, found a function call
   --> tests/runner/src/scenarios/prepared_statements.rs:577:22
    |
577 |             (Ok(()), Ok(())) => Ok(()),
    |                      ^^ not a tuple struct or tuple variant
    |
    = note: function calls are not allowed in patterns: <https://doc.rust-lang.org/book/ch19-00-patterns.html>
help: consider importing this tuple variant instead
    |
  1 + use std::result::Result::Ok;
    |

error[E0532]: expected a pattern, found a function call
   --> tests/runner/src/scenarios/prepared_statements.rs:579:14
    |
579 |             (Ok(()), Err(drop_err)) => Err(drop_err),
    |              ^^ not a tuple struct or tuple variant
    |
    = note: function calls are not allowed in patterns: <https://doc.rust-lang.org/book/ch19-00-patterns.html>
help: consider importing this tuple variant instead
    |
  1 + use std::result::Result::Ok;
    |

error[E0425]: cannot find value `col1` in this scope
   --> tests/runner/src/scenarios/prepared_statements.rs:596:72
    |
596 | ...ew(vec![Field::new(col1, DataType::Int32, false),Field::new(col2,D...
    |                       ^^^^ not found in this scope

error[E0425]: cannot find value `col2` in this scope
   --> tests/runner/src/scenarios/prepared_statements.rs:596:113
    |
596 | ... false),Field::new(col2,DataType::Int32,false)]));
    |                       ^^^^ not found in this scope

error[E0425]: cannot find value `id` in this scope
   --> tests/runner/src/scenarios/prepared_statements.rs:617:68
    |
617 | ...new(vec![Field::new(id, DataType::Int32, false),
    |                        ^^ not found in this scope
    |
help: consider importing one of these functions
    |
  1 + use crate::process::id;
    |
  1 + use std::process::id;
    |
  1 + use tokio::task::id;
    |

error[E0532]: expected a pattern, found a function call
   --> tests/runner/src/scenarios/prepared_statements.rs:751:9
    |
751 |         Ok(text) => text.to_string(),
    |         ^^ not a tuple struct or tuple variant
    |
    = note: function calls are not allowed in patterns: <https://doc.rust-lang.org/book/ch19-00-patterns.html>
help: consider importing this tuple variant instead
    |
  1 + use std::result::Result::Ok;
    |

Some errors have detailed explanations: E0425, E0532.
For more information about an error, try `rustc --explain E0425`.
error: could not compile `swanlake-runner` (bin "runner") due to 7 previous errors
warning: build failed, waiting for other jobs to finish...
```
由此可以看到该问题以及解决，但是还是发生了报错。
主要是两个问题：1.包引入的不对，2.变量使用的不对，可以看到rust编译器很友好，已经给出了对应问题的解决办法。

最后，进行整体编译
```bash
[root@poc-05-44 swanlake]# cargo build --release
   Compiling swanlake-server v0.1.0 (/data/soft/jdk/swanlake/swanlake-server)
   Compiling swanlake-runner v0.1.0 (/data/soft/jdk/swanlake/tests/runner)
    Finished `release` profile [optimized] target(s) in 18.82s
[root@poc-05-44 swanlake]# RUST_LOG=info cargo run --bin swanlake
   Compiling proc-macro2 v1.0.106
   Compiling quote v1.0.45
   Compiling unicode-ident v1.0.24
   Compiling libc v0.2.185
   Compiling cfg-if v1.0.4
   Compiling memchr v2.8.0
   Compiling once_cell v1.21.4
   Compiling bytes v1.11.1
   Compiling serde_core v1.0.228
   Compiling pin-project-lite v0.2.17
   Compiling itoa v1.0.18
   Compiling futures-core v0.3.32
   Compiling libm v0.2.16
   Compiling find-msvc-tools v0.1.9
   Compiling smallvec v1.15.1
   Compiling shlex v1.3.0
   Compiling autocfg v1.5.0
   Compiling futures-sink v0.3.32
   Compiling slab v0.4.12
   Compiling futures-task v0.3.32
   Compiling num-traits v0.2.19
   Compiling futures-io v0.3.32
   Compiling futures-channel v0.3.32
   Compiling http v1.4.0
   Compiling syn v2.0.117
   Compiling base64 v0.22.1
   Compiling zerocopy v0.8.48
   Compiling log v0.4.29
   Compiling jobserver v0.1.34
   Compiling mio v1.2.0
   Compiling socket2 v0.6.3
   Compiling cc v1.2.60
   Compiling bitflags v2.11.1
   Compiling http-body v1.0.1
   Compiling percent-encoding v2.3.2
   Compiling httparse v1.10.1
   Compiling tower-service v0.3.3
   Compiling stable_deref_trait v1.2.1
   Compiling hashbrown v0.17.0
   Compiling equivalent v1.0.2
   Compiling errno v0.3.14
   Compiling zmij v1.0.21
   Compiling ryu v1.0.23
   Compiling atomic-waker v1.1.2
   Compiling num-integer v0.1.46
   Compiling try-lock v0.2.5
   Compiling iana-time-zone v0.1.65
   Compiling want v0.3.1
   Compiling signal-hook-registry v1.4.8
   Compiling chrono v0.4.44
   Compiling num-bigint v0.4.6
   Compiling getrandom v0.2.17
   Compiling getrandom v0.3.4
   Compiling version_check v0.9.5
   Compiling indexmap v2.14.0
   Compiling tracing-core v0.1.36
   Compiling ahash v0.8.12
   Compiling serde v1.0.228
   Compiling num-complex v0.4.6
   Compiling arrow-schema v58.1.0
   Compiling tower-layer v0.3.3
   Compiling form_urlencoded v1.2.2
   Compiling hashbrown v0.16.1
   Compiling parking_lot_core v0.9.12
   Compiling pkg-config v0.3.33
   Compiling vcpkg v0.2.15
   Compiling typenum v1.19.0
   Compiling ring v0.17.14
   Compiling serde_json v1.0.149
   Compiling scopeguard v1.2.0
   Compiling litemap v0.8.2
   Compiling writeable v0.6.3
   Compiling lock_api v0.4.14
   Compiling http-body-util v0.1.3
   Compiling tokio v1.52.1
   Compiling lexical-util v1.0.7
   Compiling utf8_iter v1.0.4
   Compiling icu_normalizer_data v2.2.0
   Compiling zeroize v1.8.2
   Compiling icu_properties_data v2.2.0
   Compiling rustls-pki-types v1.14.0
   Compiling synstructure v0.13.2
   Compiling parking_lot v0.12.5
   Compiling hybrid-array v0.4.10
   Compiling rustix v0.38.44
   Compiling untrusted v0.9.0
   Compiling futures-util v0.3.32
   Compiling aho-corasick v1.1.4
   Compiling regex-syntax v0.8.10
   Compiling rustls v0.23.38
   Compiling cmov v0.5.3
   Compiling linux-raw-sys v0.4.15
   Compiling serde_derive v1.0.228
   Compiling zerofrom-derive v0.1.7
   Compiling zerocopy-derive v0.8.48
   Compiling zerofrom v0.1.7
   Compiling yoke-derive v0.8.2
   Compiling zerovec-derive v0.11.3
   Compiling tokio-macros v2.7.0
   Compiling displaydoc v0.2.5
   Compiling yoke v0.8.2
   Compiling tracing-attributes v0.1.31
   Compiling futures-macro v0.3.32
   Compiling zerotrie v0.2.4
   Compiling zerovec v0.11.6
   Compiling fnv v1.0.7
   Compiling unicode-segmentation v1.13.2
   Compiling tinystr v0.8.3
   Compiling potential_utf v0.1.5
   Compiling tracing v0.1.44
   Compiling icu_locale_core v2.2.0
   Compiling icu_collections v2.2.0
   Compiling crunchy v0.2.4
   Compiling crc32fast v1.5.0
   Compiling object v0.37.3
   Compiling regex-automata v0.4.14
   Compiling ctutils v0.4.2
   Compiling crypto-common v0.2.1
   Compiling icu_provider v2.2.0
   Compiling icu_normalizer v2.2.0
   Compiling icu_properties v2.2.0
   Compiling block-buffer v0.12.0
   Compiling half v2.7.1
   Compiling lexical-write-integer v1.0.6
   Compiling arrow-buffer v58.1.0
   Compiling lexical-parse-integer v1.0.6
   Compiling openssl-sys v0.9.113
   Compiling tokio-util v0.7.18
   Compiling arrow-data v58.1.0
   Compiling h2 v0.4.13
   Compiling rustls-webpki v0.103.12
   Compiling subtle v2.6.1
   Compiling semver v1.0.28
   Compiling tiny-keccak v2.0.2
   Compiling simd-adler32 v0.3.9
   Compiling rustix v1.1.4
   Compiling const-oid v0.10.2
   Compiling arrow-array v58.1.0
   Compiling httpdate v1.0.3
   Compiling rand_core v0.10.1
   Compiling getrandom v0.4.2
   Compiling anyhow v1.0.102
   Compiling hyper v1.9.0
   Compiling digest v0.11.2
   Compiling rustc_version v0.4.1
   Compiling crossterm v0.28.1
   Compiling idna_adapter v1.2.1
   Compiling lexical-parse-float v1.0.6
   Compiling lexical-write-float v1.0.6
   Compiling async-trait v0.1.89
   Compiling sync_wrapper v1.0.2
   Compiling cpufeatures v0.3.0
   Compiling adler2 v2.0.1
   Compiling unicode-width v0.2.2
   Compiling either v1.15.0
   Compiling ipnet v2.12.0
   Compiling tinyvec_macros v0.1.1
   Compiling linux-raw-sys v0.12.1
   Compiling hyper-util v0.1.20
   Compiling tinyvec v1.11.0
   Compiling itertools v0.14.0
   Compiling comfy-table v7.1.4
   Compiling arrow-select v58.1.0
   Compiling miniz_oxide v0.8.9
   Compiling tower v0.5.3
   Compiling lexical-core v1.0.6
   Compiling idna v1.1.0
   Compiling flatbuffers v25.12.19
   Compiling tokio-rustls v0.26.4
   Compiling webpki-roots v1.0.7
   Compiling atoi v2.0.0
   Compiling iri-string v0.7.12
   Compiling bumpalo v3.20.2
   Compiling ucd-trie v0.1.7
   Compiling typeid v1.0.3
   Compiling arrow-ord v58.1.0
   Compiling zlib-rs v0.6.3
   Compiling ar_archive_writer v0.5.1
   Compiling mime v0.3.17
   Compiling axum-core v0.5.6
   Compiling hyper-rustls v0.27.9
   Compiling pest v2.8.6
   Compiling psm v0.1.30
   Compiling zopfli v0.8.3
   Compiling arrow-cast v58.1.0
   Compiling tower-http v0.6.8
   Compiling prost-derive v0.14.3
   Compiling serde_urlencoded v0.7.1
   Compiling url v2.5.8
   Compiling const-random-macro v0.1.16
   Compiling xattr v1.6.1
   Compiling unicode-normalization v0.1.25
   Compiling flate2 v1.1.9
   Compiling chacha20 v0.10.0
   Compiling regex v1.12.3
   Compiling pin-project-internal v1.1.11
   Compiling serde_path_to_error v0.1.20
   Compiling filetime v0.2.27
   Compiling unicode-properties v0.1.4
   Compiling matchit v0.8.4
   Compiling unicode-bidi v0.3.18
   Compiling foreign-types-shared v0.1.1
   Compiling openssl v0.10.77
   Compiling foldhash v0.1.5
   Compiling axum v0.8.9
   Compiling hashbrown v0.15.5
   Compiling foreign-types v0.3.2
   Compiling stringprep v0.1.5
   Compiling pin-project v1.1.11
   Compiling tar v0.4.45
   Compiling prost v0.14.3
   Compiling zip v6.0.0
   Compiling rand v0.10.1
   Compiling reqwest v0.12.28
   Compiling const-random v0.1.18
   Compiling pest_meta v2.8.6
   Compiling hyper-timeout v0.5.2
   Compiling sha2 v0.11.0
   Compiling md-5 v0.11.0
   Compiling hmac v0.13.0
   Compiling tokio-stream v0.1.18
   Compiling openssl-macros v0.1.1
   Compiling stacker v0.1.23
   Compiling csv-core v0.1.13
   Compiling fallible-iterator v0.2.0
   Compiling byteorder v1.5.0
   Compiling native-tls v0.2.18
   Compiling erased-serde v0.4.10
   Compiling siphasher v1.0.2
   Compiling csv v1.4.0
   Compiling postgres-protocol v0.6.11
   Compiling phf_shared v0.13.1
   Compiling tonic v0.14.5
   Compiling libduckdb-sys v1.10502.0
   Compiling pest_generator v2.8.6
   Compiling arrow-ipc v58.1.0
   Compiling dlv-list v0.5.2
   Compiling hashlink v0.10.0
   Compiling arrow-string v58.1.0
   Compiling arrow-arith v58.1.0
   Compiling arrow-row v58.1.0
   Compiling hashbrown v0.14.5
   Compiling winnow v1.0.1
   Compiling heck v0.5.0
   Compiling openssl-probe v0.2.1
   Compiling simdutf8 v0.1.5
   Compiling rust_decimal v1.41.0
   Compiling paste v1.0.15
   Compiling arrow-json v58.1.0
   Compiling strum_macros v0.27.2
   Compiling ordered-multimap v0.7.3
   Compiling tonic-prost v0.14.5
   Compiling pest_derive v2.8.6
   Compiling arrow-csv v58.1.0
   Compiling postgres-types v0.2.13
   Compiling phf v0.13.1
   Compiling futures-executor v0.3.32
   Compiling recursive-proc-macro-impl v0.1.1
   Compiling num-rational v0.4.2
   Compiling num-iter v0.1.45
   Compiling serde_spanned v1.1.1
   Compiling toml_datetime v1.1.1+spec-1.1.0
   Compiling whoami v2.1.1
   Compiling encoding_rs v0.8.35
   Compiling arrayvec v0.7.6
   Compiling thiserror v2.0.18
   Compiling toml_writer v1.1.1+spec-1.1.0
   Compiling arraydeque v0.5.1
   Compiling yaml-rust2 v0.10.4
   Compiling arrow v58.1.0
   Compiling tokio-postgres v0.7.17
   Compiling num v0.4.3
   Compiling recursive v0.1.1
   Compiling serde-untagged v0.1.9
   Compiling futures v0.3.32
   Compiling strum v0.27.2
   Compiling json5 v0.4.1
   Compiling rust-ini v0.21.3
   Compiling tokio-native-tls v0.3.1
   Compiling ron v0.12.1
   Compiling prost-types v0.14.3
   Compiling convert_case v0.6.0
   Compiling sqlparser_derive v0.5.0
   Compiling thiserror-impl v2.0.18
   Compiling pathdiff v0.2.3
   Compiling cast v0.3.0
   Compiling toml_parser v1.1.2+spec-1.1.0
   Compiling fallible-iterator v0.3.0
   Compiling fallible-streaming-iterator v0.1.9
   Compiling lazy_static v1.5.0
   Compiling arrow-flight v58.1.0
   Compiling sharded-slab v0.1.7
   Compiling toml v1.1.2+spec-1.1.0
   Compiling duckdb v1.10502.0
   Compiling sqlparser v0.61.0
   Compiling postgres-native-tls v0.5.3
   Compiling uuid v1.23.1
   Compiling matchers v0.2.0
   Compiling tracing-serde v0.2.0
   Compiling tracing-log v0.2.0
   Compiling thread_local v1.1.9
   Compiling nu-ansi-term v0.50.3
   Compiling tonic-health v0.14.5
   Compiling tracing-subscriber v0.3.23
   Compiling dotenvy v0.15.7
   Compiling config v0.15.22
   Compiling swanlake-core v0.1.0 (/data/soft/jdk/swanlake/swanlake-core)
   Compiling swanlake-server v0.1.0 (/data/soft/jdk/swanlake/swanlake-server)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 3m 08s
     Running `target/debug/swanlake`
2026-04-28T09:40:31.073085Z  INFO swanlake-server/src/main.rs:22: service config:
ServerConfig { host: "0.0.0.0", port: 4214, ducklake_init_sql: None, duckdb_threads: None, checkpoint_databases: None, checkpoint_interval_hours: Some(24), checkpoint_poll_seconds: Some(300), max_sessions: Some(100), session_timeout_seconds: Some(900), session_id_mode: PeerAddr, log_format: "compact", status_enabled: true, status_host: "0.0.0.0", status_port: 4215, status_path_prefix: "", metrics_slow_query_threshold_ms: Some(5000), metrics_history_size: Some(200) }
2026-04-28T09:40:31.073712Z  INFO new: swanlake-core/src/engine/factory.rs:24: enter
2026-04-28T09:40:31.073834Z  INFO new: swanlake-core/src/engine/factory.rs:62: base init sql INSTALL ducklake; INSTALL httpfs; INSTALL aws; INSTALL postgres; LOAD ducklake; LOAD httpfs; LOAD aws; LOAD postgres;
2026-04-28T09:40:31.073942Z  INFO new: swanlake-core/src/engine/factory.rs:24: close time.busy=217µs time.idle=40.6µs
2026-04-28T09:40:31.074110Z  INFO new: swanlake-core/src/session/registry.rs:55: enter
2026-04-28T09:40:31.074231Z  INFO new: swanlake-core/src/session/registry.rs:60: session registry initialized max_sessions=100 session_timeout_seconds=900
2026-04-28T09:40:31.074313Z  INFO new: swanlake-core/src/session/registry.rs:55: close time.busy=200µs time.idle=6.73µs
2026-04-28T09:40:31.075459Z  INFO swanlake-server/src/status.rs:62: status server listening addr=0.0.0.0:4215
2026-04-28T09:40:31.075648Z  INFO swanlake-server/src/main.rs:71: starting SwanLake Flight SQL server addr=0.0.0.0:4214
2026-04-28T09:40:31.075545Z  INFO cleanup_idle_sessions: swanlake-core/src/session/registry.rs:115: enter
2026-04-28T09:40:31.075931Z  INFO cleanup_idle_sessions: swanlake-core/src/session/registry.rs:115: close time.busy=467µs time.idle=42.1µs
2026-04-28T09:45:26.440422Z  INFO get_flight_info_sql_info: swanlake-core/src/service/handlers/mod.rs:81: enter query=CommandGetSqlInfo { info: [8] }
2026-04-28T09:45:26.442936Z  INFO create_connection: swanlake-core/src/engine/factory.rs:75: enter
2026-04-28T09:45:31.075870Z  INFO cleanup_idle_sessions: swanlake-core/src/session/registry.rs:115: enter
2026-04-28T09:45:31.076118Z  INFO cleanup_idle_sessions: swanlake-core/src/session/registry.rs:115: close time.busy=266µs time.idle=47.2µs
2026-04-28T09:46:19.533252Z  INFO get_flight_info_sql_info: swanlake-core/src/service/handlers/mod.rs:81: enter query=CommandGetSqlInfo { info: [8] } session_id="127.0.0.1:60998"
2026-04-28T09:46:19.534069Z  INFO get_flight_info_sql_info: swanlake-core/src/service/handlers/mod.rs:81: enter query=CommandGetSqlInfo { info: [8] } session_id="127.0.0.1:60998"
2026-04-28T09:46:19.534566Z  INFO get_flight_info_sql_info: swanlake-core/src/service/handlers/mod.rs:81: close time.busy=2.54ms time.idle=53.1s query=CommandGetSqlInfo { info: [8] } session_id="127.0.0.1:60998"
^C^C^C^C^C2026-04-28T09:46:39.127742Z  INFO swanlake-server/src/main.rs:102: received SIGINT, initiating graceful shutdown
2026-04-28T09:46:39.129381Z  INFO swanlake-server/src/main.rs:124: server shutdown complete
2026-04-28T09:46:39.174134Z  INFO create_connection: swanlake-core/src/engine/factory.rs:91: created new DuckDB connection
2026-04-28T09:46:39.174394Z  INFO create_connection: swanlake-core/src/engine/factory.rs:75: close time.busy=72.7s time.idle=60.4µs
```

进行集成测试
```bash 
[root@poc-05-44 swanlake]# 
[root@poc-05-44 swanlake]# bash scripts/run-integration-tests.sh
warning: export-prefix has been renamed to --sh; old name is still available as an alias, but may removed in future breaking release
info: cargo-llvm-cov currently setting cfg(coverage); you can opt-out it by passing --no-cfg-coverage
```
