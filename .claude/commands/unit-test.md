Run a specific Ginkgo unit test in the go-controller directory.

The test pattern is: $ARGUMENTS

Steps:
1. Use grep/glob to find which package(s) contain a test matching this pattern
2. Run the test using: `cd go-controller && GINKGO_FOCUS="$ARGUMENTS" PKGS="<package>" make test`
3. If the test fails, show the relevant failure output
4. If the test passes, confirm success with a brief summary
