OpenTapioca
===========
[![Documentation Status](https://readthedocs.org/projects/opentapioca/badge/?version=latest)](https://opentapioca.readthedocs.io/en/latest/?badge=latest) [![Build Status](https://github.com/wetneb/opentapioca/workflows/CI/badge.svg)](https://github.com/wetneb/opentapioca/actions) [![Coverage Status](https://coveralls.io/repos/github/wetneb/opentapioca/badge.svg)](https://coveralls.io/github/wetneb/opentapioca)

OpenTapioca is a simple and fast [Named Entity Linking system](https://en.wikipedia.org/wiki/Entity_linking) for [Wikidata](https://www.wikidata.org/). It is kept synchronous with Wikidata in real time, encouraging users to improve the results of their entity linking
tasks by contributing back to Wikidata.

A live instance is running at https://opentapioca.org/. To run it on a server that is powerful enough, I would need 50€/month: [please help fund the service if you can](https://en.liberapay.com/OpenTapioca).

A [NIF endpoint](https://github.com/dice-group/gerbil/wiki/How-to-create-a-NIF-based-web-service) is available at:
* https://opentapioca.org/api/nif (only exposing the matches that are deemed good enough)
* https://opentapioca.org/api/nif?only_matching=false (also exposing all the other matches regardless of their score)

See [the docs](https://opentapioca.readthedocs.io/en/latest/) for more information about how it works and how to run it. See [the paper](https://arxiv.org/abs/1904.09131) for some more motivation about the design of the system.

OpenTapioca is released under the Apache-2.0 license.

# Set Up

Follow the instructions below to set up the OpenTapioca environment.

### Step 1: Clone the Repository
```bash
git clone https://github.com/ehrhart/opentapioca
```

### Step 2: Create a Docker Network
```bash
docker network create opentapioca-network
```

### Step 3: Set Up the Solr Server
```bash
docker run --name=opentapioca-solr --env='SOLR_JAVA_MEM=-Xms10g -Xmx10g' --volume=opentapioca-solr-data:/var/solr --volume=./configsets:/configsets --network=opentapioca-network -p 8983:8983 --restart=unless-stopped --detach=true solr:8 -c -m 8g
docker exec -it opentapioca-solr bin/solr zk -upconfig -z localhost:9983 -n tapioca -d /configsets/tapioca
```

### Step 4: Build the Application
```bash
cp settings_template.py settings.py
docker build -t opentapioca .
```

### Step 5: Prepare the Data Dump
1. Create a directory for the data dump:
   ```bash
   mkdir dump
   cd dump/
   ```
2. Download the latest Wikidata JSON dump:
   ```bash
   wget https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.json.bz2
   ```
3. Index the dump using the OpenTapioca profile:
   ```bash
   docker run --rm --detach=true --network=opentapioca-network --volume=./dump:/app/dump --volume=./profiles:/app/profiles opentapioca bash -c "bunzip2 < dump/latest-all.json.bz2 | tapioca index-dump wd_2019-02-24 - --profile profiles/human_organization_location.json"
   ```

### Step 6: Compute or Download Supporting Data
#### Option A: Use Precomputed Data
Download the precomputed data files:
```bash
wget -c https://github.com/wetneb/opentapioca/releases/download/v0.1.0/wd_2019-02-24.bow.pkl -O data/wd_2019-02-24.bow.pkl
wget -c https://github.com/wetneb/opentapioca/releases/download/v0.1.0/wd_2019-02-24.pgrank.npy -O data/wd_2019-02-24.pgrank.npy
wget -c https://github.com/wetneb/opentapioca/releases/download/v0.1.0/sample_classifier.pkl -O data/rss_istex_classifier.pkl
```

#### Option B: Compute the Data Yourself
1. Set up a Python virtual environment:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   python setup.py install
   ```
2. Train and preprocess the data:
   ```bash
   tapioca train-bow latest-all.json.bz2
   tapioca preprocess latest-all.json.bz2
   ```
3. Sort and compile the graph:
   ```bash
   sort -n -k 1 latest-all.unsorted.tsv > wikidata_graph.tsv
   tapioca compile wikidata_graph.tsv
   ```
4. Compute PageRank:
   ```bash
   tapioca compute-pagerank wikidata_graph.npz
   ```

### Step 7: Run the OpenTapioca Application
Start the OpenTapioca application:
```bash
docker run --name=opentapioca-app --env=SOLR_ENDPOINT=http://opentapioca-solr:8983/solr --volume=/home/cixty/Services/opentapioca/dump:/app/dump --volume=/home/cixty/Services/opentapioca/data:/app/data --network=opentapioca-network -p 8457:8457 --restart=unless-stopped --detach=true opentapioca
```

Open your browser and navigate to http://localhost:8457.
