## Java Performance Companion

### Ch 1

CSet: The optimal set of old gen regions to collect

When heap usage exceed IHOP, initial mark is triggered with the next young GC.

G1 Concurrent cycle: initial marking, concurrent root region scanning, concurrent marking, remarking, and cleanup
  1. Initial mark: STW (done in a young GC)
  2. Concurrent root region scanning: can and follow all references from objects in the survivor regions; Not STW
  3. Concurrent marking: multiple threads to mark the live object graph; Not STW
  4. Remarking: STW
  5. Cleanup: reclaim regions.

Concurrent cycle is critical to make decisions about what regions to include in the mixed GC.

Heap resizing cases:
  1. After full GC
  2. Based on -XX:GCTimeRatio, meaning if too much time spent in doing GC
  3. Object allocation fails
  4. Humongous object allocation fails
  5. GC request a new region to evacuate objects
