IDL = $(wildcard *.proto)
google_deps = Mgoogle/protobuf/empty.proto=github.com/gogo/protobuf/types,Mgoogle/protobuf/timestamp.proto=github.com/gogo/protobuf/types,Mgogo.proto=github.com/gogo/protobuf/gogoproto
deps = $(google_deps)

.PHONY: all
all: $(IDL)
	protoc -I=. -I=$$PROTO_INCLUDE \
		$^ \
		--gofast_out=plugins=grpc,$(deps):.
