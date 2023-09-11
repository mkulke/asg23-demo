FROM rust:1.71-slim-bullseye as builder
ARG KBS_COMMIT=81f3de7

RUN apt-get update && apt-get install -y curl gnupg2
RUN curl -L https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add -
RUN echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' > /etc/apt/sources.list.d/intel-sgx.list
RUN apt-get update && apt-get install -y \
	build-essential \
	clang \
	git \
	libsgx-dcap-quote-verify-dev \
	libtdx-attest-dev \
	libtss2-dev \
	openssl \
	pkg-config \
	protobuf-compiler
RUN curl -LO https://go.dev/dl/go1.20.7.linux-amd64.tar.gz
RUN tar -C /usr/local -xzf go1.20.7.linux-amd64.tar.gz
ENV PATH=$PATH:/usr/local/go/bin
RUN git clone https://github.com/confidential-containers/kbs.git -b main /build
WORKDIR /build
RUN git checkout $KBS_COMMIT
RUN cargo b --release -p kbs

FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl gnupg2
RUN curl -L https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add -
RUN echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' > /etc/apt/sources.list.d/intel-sgx.list
RUN apt-get update && apt-get install -y \
	libsgx-dcap-quote-verify \
	libtss2-esys-3.0.2-0 \
	libtss2-tctildr0 \
	openssl
ENV KBS_REPO_PATH=/opt/confidential-containers/kbs/repository
RUN openssl genpkey -algorithm ed25519 > /kbs.key
RUN openssl pkey -in /kbs.key -pubout -out /kbs.pem
COPY --from=builder /build/target/release/kbs /kbs
RUN mkdir -p "${KBS_REPO_PATH}/default/key"
RUN mkdir -p "${KBS_REPO_PATH}/default/credential"
# we need a default credential object for cc_kbc to work
RUN echo '{"auths":{}}' > "${KBS_REPO_PATH}/default/credential/test"
CMD ["/kbs", "--socket", "0.0.0.0:8080", "--insecure-http", "--auth-public-key", "/kbs.pem" ]
