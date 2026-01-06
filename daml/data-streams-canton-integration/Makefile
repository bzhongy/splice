# Makefile for Daml multi-package project

# Auto-discover all packages by finding daml.yaml files
MAIN_PACKAGES := $(shell find src -path "*/main/daml.yaml" -exec dirname {} \; | sort)
TEST_PACKAGES := $(shell find src -path "*/test/daml.yaml" -exec dirname {} \; | sort)
ALL_PACKAGES := $(MAIN_PACKAGES) $(TEST_PACKAGES)

.PHONY: build test clean lint

# Build everything (main + tests)
build:
	daml build --all

# Clean all build artifacts
clean:
	daml clean --all

# Run all tests
test: build
	@echo "Running all tests..."
	@for pkg in $(TEST_PACKAGES); do \
		echo ""; \
		echo "Running tests in $$pkg..."; \
		(cd $$pkg && daml test --all) || exit 1; \
	done


# Run DAML linter to ensure code quality and best practices
lint:
	@echo "Linting all packages..."
	@for pkg in $(ALL_PACKAGES); do \
		echo "Linting $$pkg..."; \
		find $$pkg/daml -name "*.daml" -exec daml damlc lint {} \; 2>/dev/null || true; \
	done
