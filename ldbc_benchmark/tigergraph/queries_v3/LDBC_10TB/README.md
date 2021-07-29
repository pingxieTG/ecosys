# Prepare LDBC SNB 10 TB data
## Table of Contents
* [Direct download](#Direct-download-(not-recommended))
* [Download one partition](#Download-one-partition-of-the-data)
   * [Pre-requisites](#Pre-requisites)
   * [Download data](#Donwload-data)
   * [Uncompress data](#Uncompress-data)
* [Run queries and updates](#Run-queries-and-updates)
* [About queries](#About-queries)

## Direct download (not recommended)
The data of 10TB is located in `gs://ldbc_snb_10k/v1/results/sf10000-compressed/runs/20210713_203448/social_network/csv/bi/composite-projected-fk/` and 30 TB in `gs://ldbc_snb_30k/results/sf30000-compressed/runs/20210728_061923/social_network/csv/bi/composite-projected-fk/`. You can use `gsutil ls` to explore the two folder
```sh
gsutil ls gs://ldbc_snb_10k/v1/results/sf10000-compressed/runs/20210713_203448/social_network/csv/bi/composite-projected-fk/
```
You should see four folders: `initial_snapshot`, `inserts`, `inserts_split`, `deletes`. The `inserts_split` is the same data as `inserts` but csv files are split into smaller CSVs.

To download the whole data using (Google Cloud SDK)[https://cloud.google.com/sdk/docs/install]. 
```sh
gsutil -m cp -r  gs://ldbc_snb_10k/v1/results/sf10000-compressed/runs/20210713_203448/social_network/csv/bi/composite-projected-fk/ .  
```
option `-m` means running using multiple threads. When using multiple machines, I recommend to only donwload one part for each machine. The procedures is in the next section.

## Download one partition of the data
### Pre-requisites
`python3` is required to run the script we will use `google-cloud-storage` package. and `gzip` and GNU `parallel` are required to uncompress the data. On CentOs, the command is
```sh
sudo yum install -y  python3-pip perl bzip2 gzip wget lynx
# install python3 google-cloud-storage package
pip3 install google-cloud-storage
# install GNU parallel
(wget -O - pi.dk/3 || lynx -source pi.dk/3 || curl pi.dk/3/ || \
   fetch -o - http://pi.dk/3 ) > install.sh
sh install.sh
```

### Download data
Use the `download_one_partition.py` to download one partition of the data. The python script in the next step requires a GCP service key in json. The data is public and open to all users, so it is no matter what the public key is. The tutorial for setting up the service key can be found on [GCP docs](https://cloud.google.com/docs/authentication/getting-started).

The usage of the script is `python3 download_one_partition.py [node index] [number of nodes]`. For a cluster of 4 nodes, you need to run the command on all of the 4 nodes and use the nodex index 0,1,2,3 for each machine. I also prefer to run in the background using `nohup`.
```sh
# on node m1
nohup python3 -u download_one_partition.py 10t 0 4  > foo.out 2>&1 < /dev/null &
```
The data location in GCS bucket is hard coded in the code. The data is downloaded to `./sf10000/`. For 30TB data, use
```sh
# download 30TB data
nohup python3 -u download_one_partition.py 30t 0 4 -d 30t  > foo.out 2>&1 < /dev/null &
```

### Decompress data
Uncompress the data on each node in parallel.
```sh
cat << EOF > uncompress.sh
cd sf10000
mv inserts_split inserts 
find . -name *.gz  -print0 | parallel -q0 gunzip 
echo 'done uncompress'
EOF
nohup sh uncompress.sh  > foo2.out 2>&1 < /dev/null &
```

We also provide a script `download_all.py` to download and decompress for all the nodes. The script require installation of `paramiko` and `scp` on the host. The usage is 
```sh
python3 download_all.py [data] [start ip addresss] [number of nodes] 
#for example
python3 download_all.py 30t 10.128.0.4 10
```

## Run queries and updates
The dataset does not have header. So do not add the option `--header`. To load the data (take ~12hr)
```sh
./driver.py load all ~/sf10000 
```
Run all the queries
```sh
./driver.py run 
```
Perform the batch update, begin date is `2012-11-29`, end date is `2012-12-31`. We perform bi reading queries every 7 days, we also add sleep factor 1. 
```sh
./driver.py refresh ~/sf10000/ -b 2012-11-29 -e 2012-12-31 -q not:17,19 -r 7 -s 1
```

The combine command in background is
```sh
nohup python3 -u ./driver.py all ~/sf10000/ -b 2012-11-29 -e 2012-12-31 -r 7 -s 1  > foo.out 2>&1 < /dev/null & 
```

## About queries
The BI queries on 10TB data is typically ~60 s and greatly depends on the parameters. We chose some parameters that are easy for use, for example if we filter the comments after an input date, we will chose a later time for the Social network graph.

BI20 have several version. The query is fast for some parameters (in this case, the non-distributed one `bi20-1.gsql` takes less time). But the query is super slow for other parameters (>200s, in this case, the distributed one `bi20-2.gsql` is better). We have not validated `bi20-3` yet.