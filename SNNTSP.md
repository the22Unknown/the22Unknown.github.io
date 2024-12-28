# Stochastic Nearest Neighbor TSP using MPI and OpenMP

> Adapted from a class project in High Performance Computing at the University of Tulsa.

For this project I chose to explore a stochastic variation of the Nearest Neighbor heuristic for the Traveling Salesman problem. This uses the Nearest Neighbor heuristic for this problem as normal but uses a degree of randomness to explore away from the heuristic. I hoped this would lead to exploring paths away from the Nearest Neighbor solution that would fare better than nearest neighbor.

## Methodology

To do this, I had a parameter for Degree of Randomness, which I called ODDS. This odds determined the likely hood of any individual step resorting to randomness rather than the heuristic. This, in combination with the standard nearest neighbor heuristic looks as follows:

```
Starting city = random start;
While there are unvisited cities,
	If( random chance (ODDS) )
		Randomly choose an unvisited city
	Else
		Choose the unvisited city closest to the last
```

To parallelize this, I took advantage of both MPI and OpenMP. MPI splits the work across several machines, and OpenMP splits the work among the cores of a single machine. In my implementation, the threads on each machine all do the same function, running through the above pseudocode as many times as possible within the time limit, while maintaining memory of the best path they've seen so far.

To time this system, I used the standard `<time.h>` functions, since I only needed accuracy to the second. My code checks the time at the end of every path, and if they've passed the time limit the code breaks from the loop and begins collecting results.

## Results

This implementation revealed several interesting things about the problem, including the effects of randomness upon both the quality of answer and the quantity of paths checked by the process.

![](./img/PathsChecked.png)

What I found was that the degree of randomness greatly affected the number of paths that the process could explore. The `rand()` function in our C implementation was slow compared to the heuristic check, so running a higher degree of randomness (leaning heavily on the `rand()` function rather than the heuristic) decreased the number of paths the process could check within the time constraints. The above graph uses the average among the individual processes on each machine to measure this value.

![](./img/BestCost.png)

The next thing I found was that the amount of randomness affected the quality of path found during the runtime. This actually shows that a large amount of randomness was harmful to the answer quality the process found, at anything above ODDS=5%. At and below ODDS=5%, the answers were very close to the non-random value. To explore this, I ran an experiment with large randomness but more time, to make up for the time issue that `rand()` applies to the process.

I did this by setting the ODDS to 90%, then increasing the time proportional to the gap in the number of paths checked. This resulted in multiplying the time constraint by 4 to attempt to 'even the odds' for the `rand()` function. By doing this, I was able to get the Average paths checked per process to 145,000, but the best cost path for the process was still at 43,080, more than 8 times the low randomness answer.

In conclusion, the Nearest Neighbor heuristic for the Traveling Salesman problem does not benefit from an aspect of randomness, but actually suffers to find a good path under the added randomness.

## Appendix A: Implementation

```c
/* Traveling Salesman Problem
 * - Implements basic Heuristics as a search method, with added randomness
 * - Multiple Nodes with multiple threads
 * 		- Nodes handle different starting cities
 * 		- Threads handle different random paths from the same starting city
 * - Returns the shortest path found in 60ish seconds
 */

#include <stdio.h>
#include <time.h>
#include <stdlib.h>
#include <limits.h>
#include <mpi.h>
#include <omp.h>

#define ODDS 2
#define FILENAME "DistanceMatrix1000_v2.csv"

int main(int argc, char* argv[]){
    // MPI Initializing
    MPI_Init(NULL, NULL);
    int commSz;
    MPI_Comm_size(MPI_COMM_WORLD, &commSz);
    int rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    time_t start;
    srand((unsigned) time(&start));
    // Start Timer
    int (*distMat)[1000] = malloc(sizeof(int[1000][1000]));
    if(distMat==NULL){
        printf("Memory Allocation Issue: Matrix");
        exit(1);
    }
    //printf("Rank: %d\n", rank);
    if(rank==0){
	    // Read Input
	    FILE* filePointer;
	    filePointer = fopen(FILENAME,"r");
	    for(int i = 0; i<1000;i++){
	        for(int j=0;j<1000;j++){
	            fscanf(filePointer, "%d,", &distMat[i][j]);
	        }
	        fscanf(filePointer, "\n");
	    }
	}
    // Search Loop
	MPI_Bcast(&(distMat[0][0]),1000*1000, MPI_INT, 0, MPI_COMM_WORLD);
	// Randomly pick a starting city
	int startCity; // = rand() % 1000;
	// distMat[A][B] = distance from A->B
	int* bestPath;
	int bestCost = INT_MAX;
	int cityCount = 0;
	#pragma omp parallel private(startCity) reduction(+:cityCount)
	{
		time_t tim;
		int threadRank = omp_get_thread_num();
		startCity = ((rand()%1000)*threadRank + rank) % 1000;
		int *currPath = calloc(1000, sizeof(int));
		currPath[0] = startCity;
		int currCost = 0;
		time(&tim);
		while(tim-start < 60){
			cityCount++;
			int *visited = calloc(1000, sizeof(int)); //Visited array
			int prev = startCity;
			currCost = 0;
			int next;
			for(int i =1;i<1000;i++){
				//Each Node on the path
				if(rand()%100<ODDS){
					// IF RANDOM
					do{
						next = rand()%1000;
					} while(visited[next]);
				} else {
					next =0;
					for(int j = 0;j<1000;j++){
						if(visited[j]) continue;
						if(distMat[prev][j] < distMat[prev][next]){ // Nearest Neighbor
							next = j;
						}
						if(distMat[prev][j]<=5){
							next = j;
							break;
						}
					}
					// ELSE HEURISTIC
				}
				// Count up stats, update visited
				currCost += distMat[prev][next];
				currPath[i] = next;
				visited[next] = 1;
				prev = next;
			}
			//Tack on final step
			currCost += distMat[prev][startCity];
			// Check path against our past paths
			#pragma omp critical
			{
				if(currCost<bestCost){
					bestCost = currCost;
					bestPath = currPath;
				}
			}
			time(&tim); //Time check before looking at another path
		}
	}//PARRALLEL
	//Argmin from MPI: MINLOC then SEND/RECV path to root
	printf("[Process %d] Searched %d paths and found best path %d\n", rank, cityCount, bestCost);
	//MINLOC
	int data_pair[2];
	data_pair[0] = bestCost;
	data_pair[1] = rank;
	int reduction_result[2];
	MPI_Allreduce(data_pair, reduction_result, 1, MPI_2INT, MPI_MINLOC, MPI_COMM_WORLD);
	if((reduction_result[1]==rank) && (reduction_result[1]!=0)){ // if ME == Loc send path to root
		printf("[Process %d] Entering send mode...\n", rank);
		MPI_Send(bestPath, 1000, MPI_INT, 0, 0, MPI_COMM_WORLD);
	}
	if(rank==0){ // if ME == Root get path from Loc
		printf("Reducton Result: [0] %d [1] %d\n", reduction_result[0], reduction_result[1]);
		if(reduction_result[1]==rank){
			printf("Best Path Found Cost: %d\n", bestCost);
		} else {
			printf("[Process %d] Entering recv mode...\n", rank);
			MPI_Status stat;
			MPI_Recv(bestPath, 1000, MPI_INT, MPI_ANY_SOURCE, 0, MPI_COMM_WORLD, &stat);
			printf("BestPathCost: %d\n", reduction_result[0]);
		}
		for(int i = 0;i<10;i++) printf("%d ", bestPath[i]);
		printf("\n");
		// Print results
	}
	// Report output
	MPI_Finalize();
	return 0;
}
```
