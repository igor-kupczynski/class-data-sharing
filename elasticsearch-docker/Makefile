.PHONY: help
help:
	@ echo "build"
	@ echo "run"


pkg/jdk-10.0.1_linux-x64_bin.rpm:
	@ wget -P pkg --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/10.0.1+10/fb4372174a714e6b8c52526dc134031e/jdk-10.0.1_linux-x64_bin.rpm

pkg: pkg/jdk-10.0.1_linux-x64_bin.rpm


.PHONY: build
build: pkg
	@ docker build -t "igor-kupczynski/elasticsearch-oss:6.2.4-jdk10" .


.PHONY: run
run:
	docker run --rm -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -it "igor-kupczynski/elasticsearch-oss:6.2.4-jdk10"
