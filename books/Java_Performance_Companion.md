## "Java Performance Companion"

### Ch 1

CSet: The optimal set of old gen regions to collect

When heap usage exceed IHOP, initial mark is triggered with the next young GC.

G1 Concurrent cycle: initial marking, concurrent root region scanning, concurrent marking, remarking, and cleanup
