.PHONY: build clean install-ksl

KSL_FILES := kessel.ksl rbac.ksl drive.ksl
OUTPUT := schema.zed

# Build ZED schema from KSL files
build: $(OUTPUT)

$(OUTPUT): $(KSL_FILES)
	ksl $(KSL_FILES)

# Install KSL compiler
install-ksl:
	go install github.com/project-kessel/ksl-schema-language/cmd/ksl@latest

# Clean generated files
clean:
	rm -f $(OUTPUT)

# Show help
help:
	@echo "Usage:"
	@echo "  make build       - Compile KSL files to schema.zed"
	@echo "  make install-ksl - Install the KSL compiler"
	@echo "  make clean       - Remove generated schema.zed"

.DEFAULT_GOAL := help
