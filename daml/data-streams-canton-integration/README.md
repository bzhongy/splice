# data-streams-canton-integration

A DAML project for Chainlink Data Streams integration with Canton.

## Development

### Prerequisites
- [Daml SDK 3.4.9](https://docs.daml.com/getting-started/installation.html)
- [Java 17](https://adoptium.net/temurin/releases/?version=17) or higher

### Installing Daml SDK

```bash
curl -sSL https://get.daml.com/ | sh
daml install 3.4.9
```

### Building and Testing
```bash
# Build all packages (main + tests)
make build

# Run all tests
make test

# Lint all DAML files for code quality and best practices
make lint

# Clean build artifacts
make clean
```

#### Manual Commands (if needed)
```bash
# Build all packages
daml build --all

# Build main package only
daml build --project-root src/main

# Build test package only  
daml build --project-root src/test

# Run tests
daml test --project-root src/test
```

### CI/CD
This project uses GitHub Actions for continuous integration:
- **Build**: Compiles DAML contracts
- **Test**: Runs all test suites
- **Lint**: Ensures code quality and DAML best practices

## References
- [DAML Smart Contracts Documentation](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/index.html)
