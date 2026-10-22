# SorobanLard Contract — Makefile
#
# Deployment runbook: after running `make deploy-testnet`, record the resulting
# contract ID, deployed Wasm hash, and deployment date in DEPLOYMENTS.md at the
# repository root. Keep DEPLOYMENTS.md in sync with every deployment.

NETWORK       = testnet
CONTRACT_NAME = sorobanlard
WASM_PATH     = target/wasm32v1-none/release/$(CONTRACT_NAME).wasm
WASM_OPT      = target/wasm32v1-none/release/$(CONTRACT_NAME).optimized.wasm

.PHONY: test build optimize deploy-testnet clean fmt check

test:
	cargo test

build:
	stellar contract build

optimize: build
	stellar contract optimize --wasm $(WASM_PATH)
	@echo "Optimized: $(WASM_OPT)"

# Deploy to testnet. After a successful deploy, update DEPLOYMENTS.md with the
# new contract ID, Wasm hash, and deployment date (see repo root DEPLOYMENTS.md).
deploy-testnet: optimize
	stellar contract deploy \
		--wasm $(WASM_OPT) \
		--source $(STELLAR_ACCOUNT) \
		--network $(NETWORK)
	@echo "Deployed. Remember to record the contract ID, Wasm hash, and date in DEPLOYMENTS.md."

fmt:
	cargo fmt

check:
	cargo check

clean:
	cargo clean
