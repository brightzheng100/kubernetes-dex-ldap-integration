.PHONY: cfssl ca req-dex req-k8s clean

CFSSL	= @env PATH=$(GOPATH)/bin:$(PATH) cfssl
JSON	= env PATH=$(GOPATH)/bin:$(PATH) cfssljson

all: cfssl ca req-dex req-k8s

cfssl:
	go get -u -tags nopkcs11 github.com/cloudflare/cfssl/cmd/cfssl
	go get -u github.com/cloudflare/cfssl/cmd/cfssljson

ca:
	mkdir -p _certs
	$(CFSSL) gencert -initca ca-csr.json | $(JSON) -bare _certs/ca

req-dex:
	$(CFSSL) gencert \
	  -ca _certs/ca.pem \
	  -ca-key _certs/ca-key.pem \
	  -config ca-config.json \
	  req-csr-dex.json | $(JSON) -bare _certs/dex

req-k8s:
	$(CFSSL) gencert \
	  -ca _certs/ca.pem \
	  -ca-key _certs/ca-key.pem \
	  -config ca-config.json \
	  req-csr-k8s.json | $(JSON) -bare _certs/k8s

clean:
	rm -rf _certs