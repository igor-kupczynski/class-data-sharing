ROOT_DIR:=$(shell dirname $(realpath $(lastword $(MAKEFILE_LIST))))

.PHONY: help
help:
	@ echo "--- Work in progress ---"

###############################################################################
# Step #0 Repack elasticsearch with openjdk10
###############################################################################
pkg/openjdk-10.0.1_linux-x64_bin.tar.gz:
	@ wget -P pkg https://download.java.net/java/GA/jdk10/10.0.1/fb4372174a714e6b8c52526dc134031e/10/openjdk-10.0.1_linux-x64_bin.tar.gz


pkg: pkg/openjdk-10.0.1_linux-x64_bin.tar.gz


.PHONY: build-jdk10
build-jdk10: pkg
	@ docker build -f Dockerfile-openjdk10 -t "ikupczynski/elasticsearch-oss:6.2.4-openjdk10" .


.PHONY: clean-cache
clean-cache:
	@ rm -rf $(ROOT_DIR)/cache/elasticsearch_appcds*
	@ echo "Nuked cache"


###############################################################################
# Step #1 Create class list
###############################################################################
cache/elasticsearch_appcds.cls:
	@ mkdir -p $(ROOT_DIR)/cache
	@ touch $(ROOT_DIR)/cache/elasticsearch_appcds.cls
	@ export ES_JAVA_OPTS="-XX:+UseAppCDS \
			-XX:DumpLoadedClassList=/app/cache/elasticsearch_appcds.cls" && \
		docker run \
			-d --name generate-class-list \
			-p 9200:9200 -p 9300:9300 \
			-e "discovery.type=single-node" \
			-v $(ROOT_DIR)/cache:/app/cache \
			--env ES_JAVA_OPTS \
			-it "ikupczynski/elasticsearch-oss:6.2.4-openjdk10"
	@ echo "Waiting until elasticsearch starts"
	@ bin/wait-on-elastic
	@ docker rm -f generate-class-list
	@ echo "Class list generated. Number of classes: "
	@ wc -l cache/elasticsearch_appcds.cls


# Workaround of the JVM error
cache/elasticsearch_appcds.cls-thin: cache/elasticsearch_appcds.cls
	@ head -n 6218 cache/elasticsearch_appcds.cls > cache/elasticsearch_appcds.cls-thin
	@ echo "Filtered the class list. Number of classes: "
	@ wc -l cache/elasticsearch_appcds.cls-thin

generate-class-list: cache/elasticsearch_appcds.cls-thin


###############################################################################
# Step #2 Dump classes
###############################################################################
cache/elasticsearch_appcds.jsa: generate-class-list
	@ touch $(ROOT_DIR)/cache/elasticsearch_appcds.jsa
	@ export ES_JAVA_OPTS="-Xshare:dump \
			-XX:+UseAppCDS \
			-XX:SharedClassListFile=/app/cache/elasticsearch_appcds.cls-thin \
			-XX:+UnlockDiagnosticVMOptions \
			-XX:SharedArchiveFile=/app/cache/elasticsearch_appcds.jsa" && \
		docker run \
			--rm --name dump-class-cache \
			-e "discovery.type=single-node" \
			-v $(ROOT_DIR)/cache:/app/cache \
			--env ES_JAVA_OPTS \
			-it "ikupczynski/elasticsearch-oss:6.2.4-openjdk10"


dump-class-cache: generate-class-list cache/elasticsearch_appcds.jsa


###############################################################################
# Step #3 Build image with the class cache
###############################################################################

CDS_IMAGE = ikupczynski/elasticsearch-oss:6.2.4-cds

.PHONY: build-cds
build-cds: dump-class-cache
	@ docker build -f Dockerfile-cds -t $(CDS_IMAGE) .



###############################################################################
# Step #4 Run without cds
###############################################################################

RUN_NO_CDS = export ES_JAVA_OPTS="-Xshare:off" && \
		docker run \
			--rm --name run-no-cds \
			-p 9200:9200 -p 9300:9300 \
			-e "discovery.type=single-node" \
			-v $(ROOT_DIR)/cache:/app/cache \
			--env ES_JAVA_OPTS \
			-it 

.PHONY: run-nocds
run-nocds:
	@ $(RUN_NO_CDS) $(CDS_IMAGE)

.PHONY: time-nocds
time-nocds:
	@ $(RUN_NO_CDS) -d $(CDS_IMAGE)
	@ echo "Timing the wait on elastic"
	@ time bin/wait-on-elastic
	@ docker rm -f run-no-cds


###############################################################################
# Step #5 Run with cds
###############################################################################

RUN_CDS = export ES_JAVA_OPTS="-Xshare:on \
			-XX:+UseAppCDS \
			-XX:SharedClassListFile=/app/cache/elasticsearch_appcds.cls-thin \
			-XX:+UnlockDiagnosticVMOptions \
			-XX:SharedArchiveFile=/app/cache/elasticsearch_appcds.jsa" && \
		docker run \
			--rm --name run-cds \
			-p 9200:9200 -p 9300:9300 \
			-e "discovery.type=single-node" \
			-v $(ROOT_DIR)/cache:/app/cache \
			--env ES_JAVA_OPTS \
			-it

.PHONY: run-cds
run-cds:
	@ $(RUN_CDS) $(CDS_IMAGE)

.PHONY: time-cds
time-cds:
	@ $(RUN_CDS) -d $(CDS_IMAGE)
	@ echo "Timing the wait on elastic"
	@ time bin/wait-on-elastic
	@ docker rm -f run-cds
