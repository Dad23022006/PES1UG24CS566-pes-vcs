Branching and Checkout

Q5.1: A branch in Git is just a file in .git/refs/heads/ containing a commit hash. Creating a branch is creating a file. Given this, how would you implement pes checkout <branch> — what files need to change in .pes/, and what must happen to the working directory? What makes this operation complex?

    Files in .pes/: The .pes/HEAD file must be updated to point to the new branch reference (e.g., ref: refs/heads/new-branch). Additionally, the Index (or staging area) must be updated to reflect the file structure and hashes of the commit that the new branch points to.

    Working Directory: The physical files on your disk must be synchronized with the tree of the target commit. This involves deleting files that don't exist in the new branch, creating files that are new, and overwriting files that have changed.

    Complexity: The operation is complex because it must be atomic and safe. If you have uncommitted changes, a simple checkout could overwrite your work. The system must perform a "three-way" check between the current working directory, the current HEAD, and the target branch to ensure no data is lost.

Q5.2: When switching branches, the working directory must be updated to match the target branch's tree. If the user has uncommitted changes to a tracked file, and that file differs between branches, checkout must refuse. Describe how you would detect this "dirty working directory" conflict using only the index and the object store.

To detect a conflict, the system compares the following:

    Index vs. Working Directory: Calculate the hash of the file currently on disk. If it differs from the hash stored in the Index, the file is "dirty" (has uncommitted changes).

    Index vs. Target Branch: Compare the hash in the Index with the hash of that same file in the Target Branch's commit tree.

    Conflict Logic: If the file is dirty and the hash in the Target Branch is different from the hash in the Index, a conflict exists. If the file is dirty but the Target Branch has the exact same version of the file as the Index, the checkout can often proceed safely.

Q5.3: "Detached HEAD" means HEAD contains a commit hash directly instead of a branch reference. What happens if you make commits in this state? How could a user recover those commits?

    What happens: Commits proceed normally, but the HEAD pointer moves forward by holding the new commit hash directly. Since no named branch (like main) is being updated, these commits are not "anchored" to any branch.

    Recovery: If the user switches back to another branch, the detached commits become "orphaned"—they are still in the object store but have no reference pointing to them. A user can recover them by using the reflog (a log of where HEAD has been) to find the lost commit hashes and then running pes branch <name> <hash> to point a new branch at them.

Garbage Collection and Space Reclamation

Q6.1: Over time, the object store accumulates unreachable objects — blobs, trees, or commits that no branch points to (directly or transitively). Describe an algorithm to find and delete these objects. What data structure would you use to track "reachable" hashes efficiently? For a repository with 100,000 commits and 50 branches, estimate how many objects you'd need to visit.

    Algorithm: A Mark-and-Sweep algorithm.

        Mark: Start at all "known roots" (all branch heads in refs/heads/, tags in refs/tags/, and the current HEAD). Recursively traverse the graph: Commits → Trees → Sub-trees/Blobs.

        Sweep: Any object in the object store not reached during the "Mark" phase is unreachable and can be deleted.

    Data Structure: A Hash Set (or a Bloom Filter for very large scales) is ideal for tracking reachable hashes with O(1) lookup.

    Estimation: You would visit all 100,000 commits. Given that each commit points to a tree, and trees point to blobs, you might visit millions of objects. However, since many commits share the same trees/blobs, you only process unique hashes once. In a repo of this size, you might track roughly 500,000 to 1,000,000 unique objects.

Q6.2: Why is it dangerous to run garbage collection concurrently with a commit operation? Describe a race condition where GC could delete an object that a concurrent commit is about to reference. How does Git's real GC avoid this?

    Danger: GC might delete an object that is "live" but not yet "reachable" via a branch pointer.

    Race Condition:

        User starts a commit. The tool creates a new Blob and writes it to the disk.

        Simultaneously, GC runs. It scans the branches, sees the new Blob isn't linked to any branch yet (because the commit hasn't finished updating the HEAD), and deletes it.

        The commit process finishes and updates the branch to point to a new Tree that expects that Blob to exist. The repository is now corrupted.

    The Fix: Git uses a grace period. By default, git gc will only prune (delete) unreachable objects that are older than a certain threshold (e.g., 2 weeks). This ensures that any "in-flight" objects created by concurrent processes are preserved.
