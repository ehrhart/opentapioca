Using Docker
======================

The following instructions demonstrate how to set up OpenTapioca using Docker.

Creating the Network
--------------

Create a dedicated Docker network for OpenTapioca services::

   docker network create opentapioca-network

Setting up Solr
--------------

Launch the Solr instance in Docker with appropriate configuration::

   docker run --name=opentapioca-solr \
      --env='SOLR_JAVA_MEM=-Xms10g -Xmx10g' \
      --volume=opentapioca-solr-data:/var/solr \
      --volume=./configsets:/configsets \
      --network=opentapioca-network \
      --publish 8983:8983 \
      --restart=unless-stopped \
      --detach=true \
      solr:8 -c -m 8g

Upload the Tapioca configset to Solr::

   docker exec -it opentapioca-solr bin/solr zk -upconfig -z localhost:9983 -n tapioca -d /configsets/tapioca

Building the Application
--------------

Prepare the application settings and build the Docker image::

   cp settings_template.py settings.py
   docker build -t opentapioca .

Preparing Data
-------

Create the necessary directories::

   mkdir -p {dump,profiles,data}

Download the Wikidata dump::

   wget -c https://dumps.wikimedia.org/wikidatawiki/entities/latest-all.json.bz2 -O dump/latest-all.json.bz2

Index the dump with the OpenTapioca profile::

   docker run \
      --rm \
      --name=opentapioca-indexing \
      --network=opentapioca-network \
      --volume=./dump:/app/dump \
      --volume=./profiles:/app/profiles \
      --env=SOLR_ENDPOINT=http://opentapioca-solr:8983/solr/ \
      --detach=true \
      opentapioca bash -c "bunzip2 < /app/dump/latest-all.json.bz2 | tapioca index-dump wd_2019-02-24 - --profile /app/profiles/human_organization_location.json"

Note that the indexing process may take several hours to complete. The `--detach` flag allows you to continue using your terminal while the process runs in the background. To check the progress, you can use `docker logs opentapioca-indexing`.

Preparing Supporting Data
--------------

You need three supporting data files for entity disambiguation. You can either download pre-computed files or generate them yourself.

Option 1: Download Pre-computed Files
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Execute these commands to download the necessary pre-computed data files::

   wget -c https://github.com/opentapioca/opentapioca/releases/download/v0.1.0/wd_2019-02-24.bow.pkl -O data/wd_2019-02-24.bow.pkl
   
   wget -c https://github.com/opentapioca/opentapioca/releases/download/v0.1.0/wd_2019-02-24.pgrank.npy -O data/wd_2019-02-24.pgrank.npy
   
   wget -c https://github.com/opentapioca/opentapioca/releases/download/v0.1.0/sample_classifier.pkl -O data/rss_istex_classifier.pkl

Option 2: Generate Your Own Data
^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you prefer to compute these files yourself with your own parameters:

1. Generate a bag of words model::
   
   docker run --rm --network=opentapioca-network --volume=./dump:/app/dump --volume=./data:/app/data opentapioca tapioca train-bow /app/dump/latest-all.json.bz2 -o /app/data/wd_2019-02-24.bow.pkl

2. Preprocess the dump::

   docker run --rm --network=opentapioca-network --volume=./dump:/app/dump --volume=./data:/app/data opentapioca tapioca preprocess /app/dump/latest-all.json.bz2 -o /app/data/latest-all.unsorted.tsv

3. Sort the preprocessed dump::

   sort -n -k 1 ./data/latest-all.unsorted.tsv > ./data/wikidata_graph.tsv

4. Convert the sorted dump into Numpy sparse adjacency matrix format::
   
   docker run --rm --network=opentapioca-network --volume=./data:/app/data opentapioca tapioca compile /app/data/wikidata_graph.tsv -o /app/data/wikidata_graph.npz

5. Compute PageRank scores::
   
   docker run --rm --network=opentapioca-network --volume=./data:/app/data opentapioca tapioca compute-pagerank /app/data/wikidata_graph.npz -o /app/data/wd_2019-02-24.pgrank.npy

6. Train the classifier::

   docker run --rm --network=opentapioca-network --volume=./data:/app/data opentapioca tapioca train-classifier -c my_solr_collection -b /app/data/wd_2019-02-24.bow.pkl -p /app/data/wd_2019-02-24.pgrank.npy -d /app/data/merged_RSS-500_and_istex_train.ttl -o /app/data/my_classifier.pkl

For more detailed instructions on generating custom data, consult the full documentation.

Running the Application
--------------

Launch the OpenTapioca application::

   docker run --name=opentapioca-app \
      --env=SOLR_ENDPOINT=http://opentapioca-solr:8983/solr \
      --volume=./dump:/app/dump \
      --volume=./data:/app/data \
      --network=opentapioca-network \
      --publish 8457:8457 \
      --restart=unless-stopped \
      --detach=true \
      opentapioca

The service will be available at http://localhost:8457.