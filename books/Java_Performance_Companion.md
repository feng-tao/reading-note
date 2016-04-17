## Java Performance Companion

## Not fully understand concepts

1. Find grain PRT
2. Pre-write barrier
3. PTAMS / NTAMS / Concurrent marking
4. Concurrent Threads uses 'finger' pointer optimization to claim G1 region(same as CMS)
5. RSet scrubbing
6. RSet coarsen

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


### Ch 2

Young Gen Sizing:
  1. -XX:G1NewSizePercent (5%)
  2. -XX:G1MaxNewSizePercent (60%)
  3. -XX:MaxGCPauseMillis

Survivor Sizing:
  1. -XX:TargetSurvivorRatio

Humongous region
  1. JDK8u40: Humongous region could be reclaimed in young GC

Mixed GC:
  1. Determined by IHOP
  2. Collect young gen and a few candidate old gen
  3. G1 concurrent cycle has fewer stages than multistaged concurrent cycle in CMS
  4. -XX:+G1MixedGCCountTarget(default 8, max mixed collections that will happen after a concurrent cycle) and -XX:G1HeapWastePercent(default 5, once dead obj hit this % mixed GC triggered )

CSet:
  1. -XX:G1MixedGCLiveThresholdPercet: default 85; liveness threshold and set limit to exclude most expensive old region from the CSet of mixed GC
  2. -XX:G1OldCSetRegionThresholdPercent: default 10; set the max limit on the number of old regions that can be collected per mixed collection pause

RSet:
  1. Data structure to help maintain and track references into its own unit of collection; keep track of which outside region has references into its own region
  2. RSet is maintained in two cases: old-to-young references; old-to-old references
  3. RSet also keep track of Popularity of regions: sparse, fine, and coarse; Each granular level has a per-region-table(PRT).

Card:
  1. Each g1 region sub-divides into chunks(card): each card is 512Byte
  2. A global card table is maintained for all the cards
  3. A sparse PRT is a hash table of card indices;

Write barrier:
  1. CMS/Parallel old write barrier is based on Urs Holzle's fast write barrier
  2. G1 uses pre-write barrier(Yuasa's SATB)

Concurrent refinement threads
  1. for maintaining RSets by scanning the logged cards in the filled log buffers(dirty card queue) then updating RSet of that regions
  2. Default same as ParallelGCThreads
  3. Once dirty card queue is full, it is retired and added to global list. The concurrent refinement threads will always process the retired dirty card queue. If concurrent refinement threads could not keep up mutator, mutator will be stopped
  4. -XX:G1ConcRefinementRedZone, -XX:G1ConcRefinementYellowZone, -XX:G1ConcRefinementGreenZone,

Concurrent marking
  1. maintain two bitmaps(previous, next) which holds the complete marking information
  2. Steps
    1. initial marking: Set NTAMS for each region to the current TOP of the region(STW, done in young GC)
    2. Root region scanning: For objects copied to survivor region during initial marking phase are considered marking roots; G1GC starts scanning survivor regions; (Concurrent)
    3. Remark: drains any remaining SATB log buffer and processes any update and traverses any unvisited live objects and reference processing(STW, -XX:ParallelGCThreads)
    4. Cleanup: Two marking bitmaps swap; identify completely free regions; sort heap to identify old regions for mixed GC(sort based on region liveness); RSet scrubbing
  3. -XX:ConcGCThreads(default is 1/4 of ParallelGCThreads)

Evacuation failures
  1. Fail to find region when trying to copy live objects from young region / old region

### Ch 3

Monitoring flags for GC
  1. -XX:+PrintGCDetails
  2. -XX:+G1SummarizeRSetStats(diagnostic, -XX:+UnblockDiagnosticVMOptions)
  3. -XX:+PrintAdaptiveSizePolicy

G1 YoungGC:
  1. -XX:ParallelGCThreads; 'GC Worker End' - 'GC Worker Start'
  2. Steps(Shown in GC Log):
    # All the following steps are parallel
    1. 'Ext Root Scanning': root regions(VM data structures,JNI thread handles, hardware registers, global variables, thread stack roots) are scanned;(High 'Diff' suggests high variances which show work is not balanced)
    2. 'Update RS': update regions with dirty cards; finish the unfinished work by concurrent refinement threads.
      - The target time of updating RSets is 10% of MaxGCPauseMillis(G1RsetUpdatingPauseTimePercent); decrease this tunable will result in increasing concurrent work.
    3. 'Scan RS': Scanned for references into CSet regions
      - More references to the region, higher the scan time
    4. 'Code Root Scanning'
    5. 'Object Copy'
      - G1 use copy times as a hint to predict time to copy a single region
    6. 'Termination'
      - Terminate the worker thread after checker other GC worker's queue
    7. 'GC Worker'
    # All the following steps are serial
    1. Code Root Fixup
    2. Code Root Purge
    3. Clear CT
    4. other
      1. Choose CSet
      2. Ref proc
      3. Ref Enq
      4. Redirty cards
      5. Humongous reclaim
      6. Free CSet

G1 Concurrent cycle:
  1. IHOP
  
