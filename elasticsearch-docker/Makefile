.PHONY: help
help:
	@ echo "build"
	@ echo "run"

pkg/openjdk-10.0.1_linux-x64_bin.tar.gz:
	@ wget -P pkg https://download.java.net/java/GA/jdk10/10.0.1/fb4372174a714e6b8c52526dc134031e/10/openjdk-10.0.1_linux-x64_bin.tar.gz 

pkg: pkg/openjdk-10.0.1_linux-x64_bin.tar.gz

.PHONY: build
build: pkg
	@ docker build -t "igor-kupczynski/elasticsearch-oss:6.2.4-jdk10" .


.PHONY: run
run:
	docker run --rm -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -it "igor-kupczynski/elasticsearch-oss:6.2.4-jdk10"

.PHONY: time
time:
	@ docker run -d --name "foo" --rm -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -it "igor-kupczynski/elasticsearch-oss:6.2.4-jdk10"
	@ echo "Timing the wait on elastic"
	@ time bin/wait-on-elastic
	@ docker rm -f foo
