.PHONY: cfssl cfssljson ca req-dex req-k8s clean

all: cfssl cfssljson ca req-dex req-k8s

cfssl:
	go install github.com/cloudflare/cfssl/cmd/cfssl@v1.6.4

cfssljson:
	go install github.com/cloudflare/cfssl/cmd/cfssljson@v1.6.4

ca: cfssl cfssljson
	mkdir -p _certs
	cfssl gencert -initca ca-csr.json | cfssljson -bare _certs/ca

req-dex: cfssl cfssljson
	cfssl gencert \
	  -ca _certs/ca.pem \
	  -ca-key _certs/ca-key.pem \
	  -config ca-config.json \
	  req-csr-dex.json | cfssljson -bare _certs/dex

req-k8s: cfssl cfssljson
	cfssl gencert \
	  -ca _certs/ca.pem \
	  -ca-key _certs/ca-key.pem \
	  -config ca-config.json \
	  req-csr-k8s.json | cfssljson -bare _certs/k8s

clean:
	rm -rf _certs