# Makefile
.PHONY: help
help:
	@ echo "# Elasticsearch and Class Data Sharing (CDS) experiment"
	@ echo
	@ echo "build-jdk10         - repackage elasticsearch-oss with openjdk10"
	@ echo 
	@ echo "clean-cache         - nuke the class list and cache"
	@ echo "generate-class-list - list classes used by elasticsearch"
	@ echo "dump-class-cache    - dump the classes from the list to cache file"
	@ echo "build-cds           - repackge the class cache in the elasticsearch docker container"
	@ echo
	@ echo "run-{cds/nocds}     - run elasticsearch with/without class data sharing"
	@ echo

pkg/openjdk-10.0.1_linux-x64_bin.tar.gz:
	@ wget -P pkg https://download.java.net/java/GA/jdk10/10.0.1/fb4372174a714e6b8c52526dc134031e/10/openjdk-10.0.1_linux-x64_bin.tar.gz

pkg: pkg/openjdk-10.0.1_linux-x64_bin.tar.gz

.PHONY: build-jdk10
build-jdk10: pkg
	@ docker build -f Dockerfile-openjdk10 -t "ikupczynski/elasticsearch-oss:6.2.4-openjdk10" .

CWD := $(shell pwd)

.PHONY: clean-cache
clean-cache:
	@ rm -rf cache/elasticsearch_appcds*
	@ echo "Nuked cache"

cache/elasticsearch_appcds.cls:
	@ touch cache/elasticsearch_appcds.cls
	@ export ES_JAVA_OPTS="-XX:+UseAppCDS \
			-XX:DumpLoadedClassList=/app/cache/elasticsearch_appcds.cls" && \
		docker run \
			-d --name generate-class-list \
			-p 9200:9200 -p 9300:9300 \
			-e "discovery.type=single-node" \
			-v $(CWD)/cache:/app/cache \
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

cache/elasticsearch_appcds.jsa: generate-class-list
	@ touch cache/elasticsearch_appcds.jsa
	@ export ES_JAVA_OPTS="-Xshare:dump \
			-XX:+UseAppCDS \
			-XX:SharedClassListFile=/app/cache/elasticsearch_appcds.cls-thin \
			-XX:+UnlockDiagnosticVMOptions \
			-XX:SharedArchiveFile=/app/cache/elasticsearch_appcds.jsa" && \
		docker run \
			--rm --name dump-class-cache \
			-e "discovery.type=single-node" \
			-v $(CWD)/cache:/app/cache \
			--env ES_JAVA_OPTS \
			-it "ikupczynski/elasticsearch-oss:6.2.4-openjdk10"

dump-class-cache: generate-class-list cache/elasticsearch_appcds.jsa

CDS_IMAGE = ikupczynski/elasticsearch-oss:6.2.4-cds

.PHONY: build-cds
build-cds: dump-class-cache
	@ docker build -f Dockerfile-cds -t $(CDS_IMAGE) .

RUN_NO_CDS = export ES_JAVA_OPTS="-Xshare:off \
			-Xlog:class+load:file=/app/logs/classload-nocds.log " && \
		docker run \
			-e "discovery.type=single-node" \
			-v $(CWD)/cache:/app/cache \
			-v $(CWD)/logs:/app/logs \
			--env ES_JAVA_OPTS \
			-it 

.PHONY: clean-nocds-logs
clean-nocds-logs:
	@ rm -f logs/classload-nocds.log*

.PHONY: run-nocds
run-nocds: clean-nocds-logs
	@ $(RUN_NO_CDS) --rm $(CDS_IMAGE)

# `time-nocds` given mostly for illustration, it is not acurate
.PHONY: time-nocds
time-nocds:
	@ $(RUN_NO_CDS) -p 9200:9200 --name run-no-cds -d $(CDS_IMAGE)
	@ echo "Timing the wait on elastic"
	@ time bin/wait-on-elastic
	@ docker rm -f run-no-cds

RUN_CDS = export ES_JAVA_OPTS="-Xshare:on \
			-Xlog:class+load:file=/app/logs/classload-cds.log \
			-XX:+UseAppCDS \
			-XX:SharedClassListFile=/app/cache/elasticsearch_appcds.cls-thin \
			-XX:+UnlockDiagnosticVMOptions \
			-XX:SharedArchiveFile=/app/cache/elasticsearch_appcds.jsa" && \
		docker run \
			-e "discovery.type=single-node" \
			-v $(CWD)/cache:/app/cache \
			-v $(CWD)/logs:/app/logs \
			--env ES_JAVA_OPTS \
			-it

.PHONY: clean-cds-logs
clean-cds-logs:
	@ rm -f logs/classload-cds.log*

.PHONY: run-cds
run-cds:
	@ $(RUN_CDS) --rm $(CDS_IMAGE)

# `time-cds` given mostly for illustration, it is not acurate
.PHONY: time-cds
time-cds:
	@ $(RUN_CDS) -p 9200:9200 --name run-cds -d $(CDS_IMAGE)
	@ echo "Timing the wait on elastic"
	@ time bin/wait-on-elastic
	@ docker rm -f run-cds
