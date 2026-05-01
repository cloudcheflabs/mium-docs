# Install

Mium is delivered as a single self-contained `tar.gz` archive: `mium-VERSION.tar.gz`. Everything Mium needs at runtime — JARs, Admin UI bundle, Python export scripts and their wheels, default configs, and the bundled Ontul SDK — is inside the archive.

## Layout After Unpacking

```bash
tar xzf mium-1.0.0.tar.gz
cd mium-1.0.0
```

```
mium-1.0.0/
├── bin/                 # start/stop scripts
├── conf/                # mium.properties, logback.xml, jvm.conf, zk/
├── data/                # RocksDB stores (created on first run)
├── lib/                 # *.jar  +  python/ wheels
├── logs/                # rotated logs
├── admin-ui/            # Admin UI bundle, served by Master at /
├── python/              # export_xlsx.py, export_pdf.py, export_pptx.py
└── ontul-sdk/           # Bundled Ontul SDK for REMOTE_JAVA / REMOTE_PYTHON
    ├── java/lib/*.jar
    └── python/ontul/
```

The launcher scripts set `MIUM_HOME` to the unpacked directory automatically — you don't need to export it yourself.

## Set the Master Key

Mium needs a 32+ character secret to derive its KMS root key. Set it **before** the first Master boot, and use the **same value** on every Master and Worker in the cluster.

```bash
export MIUM_MASTER_KEY="<paste-the-32-char-secret>"
```

Operators typically inject this from a secret manager into the systemd unit / Docker env / Kubernetes Secret that runs Mium. Once a cluster has been seeded with a master key, that key cannot be changed without re-encrypting every store — treat it like a database master password.

The variable name is configurable via `mium.kms.master.key.env` if your environment requires a different name.

## Edit `conf/mium.properties`

The defaults assume a single-node, single-host development install. Edit the values that point at your real infrastructure before the first start:

```properties
# ZooKeeper ensemble
mium.zk.connect = zk1.internal:2181,zk2.internal:2181,zk3.internal:2181

# Master & Worker bind addresses (use 0.0.0.0 to listen on all interfaces)
mium.master.host           = 0.0.0.0
mium.master.admin.port     = 8090
mium.master.internal.port  = 19099
mium.worker.host           = 0.0.0.0
mium.worker.internal.port  = 19098

# NeorunBase (Memory / Prompt / Embedding)
mium.neorunbase.jdbc.url   = jdbc:postgresql://nb.internal:5432/neorunbase?preferQueryMode=simple
mium.neorunbase.username   = mium
mium.neorunbase.password   = <secret>
mium.neorunbase.schema     = mium

# Default S3-compatible store for export ciphertext
mium.tempfile.s3.endpoint  = https://s3.internal
mium.tempfile.s3.region    = us-east-1
mium.tempfile.s3.bucket    = mium-tempfile
mium.tempfile.s3.path.style = true

# Bootstrap admin (rotate after first login)
mium.iam.admin.user        = admin
mium.iam.admin.password    = admin
```

The full property reference lives under [Configuration](../configuration/reference.md).

## Tune the JVM (Optional)

`conf/jvm.conf` is read line-by-line at start time and prepended to the JVM command line. Operators typically pin heap sizes here:

```
-Xms4g
-Xmx4g
-XX:MaxDirectMemorySize=4g
-XX:+UseG1GC
-XX:+UseStringDeduplication
--add-opens=java.base/java.nio=ALL-UNNAMED
```

The same file is used by both Master and Worker, so set it to the larger of the two. Per-role overrides go in the `JAVA_OPTS` environment variable when launching.

## Verify the Layout

```bash
ls bin/                  # start-master.sh, start-worker.sh, stop-*, ...
ls lib/*.jar | wc -l     # 50+ JARs
ls python/*.py           # 3 export scripts
ls admin-ui/index.html   # Admin UI bundle present
```

If any of these are missing, the archive is incomplete — re-extract from a known-good copy before proceeding.

## Next

- [Configuration](../configuration/reference.md) — every supported property, with defaults.
- [Run Mium](running.md) — start the Master and Worker, log in, verify health.
- [HA Cluster](ha-cluster.md) — multi-Master, multi-Worker deployment.
