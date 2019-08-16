---
title: "Mult-Threaded Web Server implemented in C"
date: 2018-12-10
tags: [Multi-Threading]
exceprt: "Multi-Threaded Web Server implemented in C, using threading techniques."
---

Multi-Threaded web server using POSIX threads (pthreads) in C to learn about thread programming and synchronization methods. Web server is able to handle any file type: HTML, GIF, JPEG, TXT, etc. and of any arbitrary size. Handles a limited portion of the HTTP web protocol (namely, the GET command to fetch a web page / files). 

Server is composed of two types of threads: dispatcher threads and worker threads. The purpose of the dispatcher threads is to repeatedly accept an incoming connection, read the client request from the connection, and place the request in a queue. We will assume that there will only be one request per incoming connection. The purpose of the worker threads is to monitor the request queue, retrieve requests and serve the request’s result back to the client. The request queue is a bounded buffer and will need to be properly synchronized (using CVs).

<h2> Thread Pool </h2>
Server creates a static/fixed pool of dispatcher and worker threads when the server starts. The dispatcher thread pool size should be num_dispatch and the worker thread pool size should be num_workers.

<h2> Incoming Requests </h2>
An HTTP request has the form: GET /dir1/subdir1/.../target_file HTTP/1.1 where /dir1/subdir1/.../ is assumed to be located under your web tree root location. Our get_request() automatically parses this for you and gives you the /dir1/subdir1/.../target file portion and stores it in the filename parameter. Your web tree will be rooted at a specific location, specified by one of the arguments to your server (See section 8 for details). For example, if your web tree is rooted at /home/user/joe/html, then a request for /index.html would map to /home/user/joe/html/index.html. You can chdir into the Web root to retrieve files using relative paths.

<h2> Caching </h2>
To improve runtime performance, we implement caching which stores cache entries in memory for faster access. When a worker serves a request, it will look up the request in the cache first. If the request is in the cache (Cache HIT), it will get the result from the cache and return it to the user. If the request is not in the cache (Cache MISS), it will get the result from disk as usual, put the entry in the cache and then return result to the user. The cache size can be defined by an argument and you need to log information about the cache (HIT or MISS) with time (see section 7,8 for more details). How to implement caching is totally up to you. You can implement a simple cache replacement policy like random policy or FIFO policy to choose an entry to evict when the cache is full.

<h2> Server Code:</h2>
```c++
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <pthread.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/time.h>
#include <fcntl.h>
#include <sys/time.h>
#include <time.h>
#include "util.h"
#include <stdbool.h>
#include <unistd.h>

#define MAX_THREADS 100
#define MAX_queue_len 100
#define MAX_CE 100
#define INVALID -1
#define BUFF_SIZE 1024

/******************************** Struct Defs *********************************/
// array implementation
typedef struct {
    int fd;
    char *request;
} request_t;

typedef struct {
    request_t *queue;
    int front, rear;
    int max_len;
} request_queue;

struct cache_entry {
    int frequency;
    char *request;
    char *content;
};

struct cache {
    struct cache_entry *cache_entries;
    int len, max_len;
};

/******************************** Global Vars *********************************/
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t some_content_cv = PTHREAD_COND_INITIALIZER;
pthread_cond_t free_slot_cv = PTHREAD_COND_INITIALIZER;
pthread_mutex_t log_lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t cache_lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cache_access_cv = PTHREAD_COND_INITIALIZER;


request_queue queue;
struct cache cache;
int id_arr[MAX_THREADS];
char *path;

/******************************** Dynamic Pool Code ***************************/
// Extra Credit: This function implements the policy to change the worker thread pool dynamically
// depending on the number of requests
void *dynamic_pool_size_update(void *arg) {
    while (1) {
        // Run at regular intervals
        // Increase / decrease dynamically based on your policy
    }
}

/******************************** LFU Cache Code **********************************/
// We implemented a LFU Cache

// Function to check whether the given request is present in cache
char *is_in_cache(char *request) {

    // Iterate through cache entries to determine whether or not request is in cache or not
    for(int i = 0; i < cache.len; i++){
      if (strcmp(cache.cache_entries[i].request, request) == 0) {
        cache.cache_entries[i].frequency = cache.cache_entries[i].frequency + 1;
        return cache.cache_entries[i].content;
      }
    }
    return NULL;
}

// Function to add the request and its file content into the cache
void add_into_cache(char *mybuf, char *memory) {
    // It should add the request at an index according to the cache replacement policy
    // Make sure to allocate/free memeory when adding or replacing cache entries

    struct cache_entry entry = {.request = mybuf, .content = memory, .frequency = 0};

    // If cache is not filled
    if (cache.len < cache.max_len) {
        cache.cache_entries[cache.len] = entry;
        cache.len++;
    } else {

        // Find least frequently used index, to replace.
        int min_index_ = 0;
        int min_value_ = cache.cache_entries[0].frequency;
        for(int i = 1; i < cache.max_len; i++){
          if(min_value_ > cache.cache_entries[i].frequency){
            min_index_ = i;
            min_value_ = cache.cache_entries[i].frequency;
          }
        }
        cache.cache_entries[min_index_] = entry;
    }
}

// clear the memory allocated to the cache
void delete_cache() {
    free(&cache);
}


// filesize method credit to Michael Potter found on stack overflow :
// https://stackoverflow.com/questions/6537436/how-do-you-get-file-size-by-fd
int fileSize(int fd) {
struct stat s;
if (fstat(fd, &s) == -1) {
  int saveErrno = errno;
  return(-1);
}
return(s.st_size);
}

// Function to initialize the cache
void init_cache(int entries) {
    // Allocating memory and initializing the cache array
    cache.cache_entries = (struct cache_entry *) malloc(sizeof(struct cache_entry) * entries);
    cache.max_len = entries;
    cache.len = 0;
}

// Function to open and read the file from the disk into the memory
// Add necessary arguments as needed
char *read_from_disk(char *request) {
    // Open and read the contents of file given the request
    int fd = open(request, O_RDONLY);
    if (fd < 0){
     printf("FDS ARE FUCKED");
	   close(fd);
	   return NULL;
    }

    int size = fileSize(fd);
    fd = open(request, O_RDONLY);
    char *mbuf = (char *) malloc(size);
    ssize_t bytes_read = read(fd, mbuf, size);
    if (bytes_read < 0) return NULL;
    close(fd);
    return mbuf;
}


char *get_from_cache(char *request, int * hitbuf) {
    pthread_mutex_lock(&cache_lock);
    // Checks to see if contents are in cache
    char *content = is_in_cache(request);
    // hit pointer, used to determine hit or miss
    *hitbuf = 1;
    if (content == NULL) {
        *hitbuf = 0;
        content = read_from_disk(request);
        add_into_cache(request, content);
    }
    pthread_mutex_unlock(&cache_lock);
    return content;
}


/******************************** Queue Code **********************************/
void init_queue(int entries) {
    //  Allocate memory and initialize queue
    queue.queue = (request_t *) malloc(sizeof(request_t) * entries);
    queue.front = queue.rear = 0;
    queue.max_len = entries;
}

int is_queue_empty() {
    return queue.front == queue.rear;
}

int is_queue_full() {
    return ((queue.rear + 1) % queue.max_len) == queue.front;
}

void enqueue(request_t request) {
    pthread_mutex_lock(&lock);
    // loop to make sure queue is not full
    while (is_queue_full()) {
        pthread_cond_wait(&free_slot_cv, &lock);
    }
    // add it to queue and increment slot
    queue.queue[queue.rear] = request;
    queue.rear = (queue.rear + 1) % queue.max_len;

    pthread_mutex_unlock(&lock);
    pthread_cond_signal(&some_content_cv);
}

request_t dequeue() {
    pthread_mutex_lock(&lock);
    // loop to make sure queue is not empty
    while (is_queue_empty()) {
        pthread_cond_wait(&some_content_cv, &lock);
    }

    // add it to queue and increment slot
    request_t request = queue.queue[queue.front];
    //queue.queue[queue.front] = NULL;
    queue.front = (queue.front + 1) % queue.max_len;

    pthread_mutex_unlock(&lock);
    pthread_cond_signal(&free_slot_cv);

    return request;
}

/******************************** Utilities ***********************************/
// Function to get the content type from the request
char *getContentType(char *mybuf) {
    // Should return the content type based on the file type in the request
    // (See Section 5 in Project description for more details)
    if (strstr(".htm", mybuf) != NULL) return "text/html";
    else if (strstr(".jpg", mybuf) != NULL) return "image/jpeg";
    else if (strstr(".gif", mybuf) != NULL) return "image/gif";
    return "text/plain";
}

// This function returns the current time in microseconds
long getCurrentTimeInMicro() {
    struct timeval curr_time;
    gettimeofday(&curr_time, NULL);
    return curr_time.tv_sec * 1000000 + curr_time.tv_usec;
}


/******************************** Dispatcher Code *****************************/
// Function to receive the request from the client and add to the queue
void *dispatch(void *arg) {
    while (1) {
        // accept client connection
        int fd = accept_connection();
	       if (fd >= 0) {  // ignore negative FDs

            // create new request
            request_t new_request;
            new_request.request = (char *) malloc(sizeof(char) * BUFF_SIZE);

            // get request from client
            if (get_request(fd, new_request.request) == 0) { // ignore if faulty
		            new_request.fd = fd;
                char *temp = strdup(new_request.request);
                strcpy(new_request.request, ".");
                strcat(new_request.request,temp);
                enqueue(new_request);
            }
        }
    }
    return NULL;
}

/******************************** Worker Code *********************************/
// Function to retrieve the request from the queue, process it and then return a result to the client
void *worker(int threadid) {
  int requests_handled = 0;
  int * hitbuf = malloc (sizeof(int));
    while (1) {
        // Start recording time
        int start_time = getCurrentTimeInMicro();

        // Get the request from the queue
        request_t retrieved_request = dequeue();

        // Get the data from the disk or the cache
	      char *content = get_from_cache(retrieved_request.request, hitbuf);
        requests_handled = requests_handled + 1;

        // Stop recording the time
        int end_time = getCurrentTimeInMicro();

        char * sprinter = malloc(BUFF_SIZE);

        // Set up log file, and terminal output
        int fd = open(retrieved_request.request, O_RDONLY);
        if (fd < 0){
          perror("Failed to open fd");
    	   close(fd);
    	   return NULL;
        }
        int size = fileSize(fd);
        int retfd = retrieved_request.fd;
        int serve_time = end_time - start_time;
        if (*hitbuf == 1) {
          sprintf(sprinter, "[%d][%d][%d][%s][%d][%dms][%s]\n", threadid, requests_handled, retfd, retrieved_request.request, size, serve_time, "HIT");
        } else {
          sprintf(sprinter, "[%d][%d][%d][%s][%d][%dms][%s]\n", threadid, requests_handled, retfd, retrieved_request.request, size, serve_time, "MISS");
        }
        int leng = strlen(sprinter);
        if (leng < 200){
          printf("%s\n", sprinter);
          pthread_mutex_lock(&log_lock);
          int wfd = open("web_server_log.log", O_APPEND | O_CREAT | O_RDWR);
          if (wfd < 0){
            perror("cannot open log file");
          }
          int bytes_write = write(wfd, sprinter, leng);
          if ((bytes_write) < 0) {
            perror ("writing to log file");
          }
          pthread_mutex_unlock(&log_lock);
        }

        // return the result
        if (content != NULL) {
            return_result(retrieved_request.fd, getContentType(retrieved_request.request), content, size);
        }
        return_error(retrieved_request.fd, retrieved_request.request);
    }
}

/******************************** Main ****************************************/
int main(int argc, char **argv) {

    // Error check on number of arguments
    if (argc != 8) {
        printf("usage: %s port path num_dispatcher num_workers dynamic_flag queue_length cache_size\n", argv[0]);
        return -1;
    }

    // Get the input args
    int port = (int) strtol(argv[1], NULL, 10);
    char *path = (char *) malloc(sizeof(argv[2]));
    strcpy(path, argv[2]);
    int num_dispatcher = (int) strtol(argv[3], NULL, 10);
    int num_workers = (int) strtol(argv[4], NULL, 10);
    int dynamic_flag = (int) strtol(argv[5], NULL, 10);
    int qlen = (int) strtol(argv[6], NULL, 10);
    int cache_entries = (int) strtol(argv[7], NULL, 10);

    // perform error checks on the input arguments (set default values if invalid)
    if (port < 1025 || port > 65535) {
        fprintf(stderr, "Provided port %d is invalid. Using default port 1025.\n", port);
        port = 1025;
    }
    if (num_dispatcher < 1 || num_dispatcher > MAX_THREADS) {
        fprintf(stderr, "Provided number of dispatcher threads %d is invalid. Using default %d.\n", num_dispatcher,
                MAX_THREADS);
        num_dispatcher = MAX_THREADS;
    }
    if (num_workers < 1 || num_workers > MAX_THREADS) {
        fprintf(stderr, "Provided number of worker threads %d is invalid. Using default %d.\n", num_workers,
                MAX_THREADS);
        num_workers = MAX_THREADS;
    }
    if (dynamic_flag != 0 && dynamic_flag != 1) {
        fprintf(stderr, "Provided dynamic flag %d is invalid. Using default 0.\n", dynamic_flag);
        dynamic_flag = 0;
    }
    if (qlen < 1 || qlen > MAX_queue_len) {
        fprintf(stderr, "Provided length of request queue %d is invalid. Using default %d.\n", qlen, MAX_queue_len);
        qlen = MAX_queue_len;
    }
    if (cache_entries < 1 || cache_entries > MAX_CE) {
        fprintf(stderr, "Provided size of cache %d is invalid. Using default %d.\n", cache_entries, MAX_CE);
        cache_entries = MAX_CE;
    }

    // Change the current working directory to server root directory
    chdir(path);

    // Start the server and initialize cache and queue
    init(port);
    init_cache(cache_entries);
    init_queue(qlen);

    // Create dispatcher and worker threads
    pthread_t *workers = malloc(sizeof(pthread_t) * num_workers);
    for (int i = 0; i < num_workers; i++) pthread_create(workers+i, NULL, worker, i);

    pthread_t *dispatchers = malloc(sizeof(pthread_t) * num_dispatcher);
    for (int i = 0; i < num_dispatcher; i++) pthread_create(dispatchers+i, NULL, dispatch, NULL);

    // Join dispatcher and worker threads
    int error;
    for(int i = 0; i < num_workers; i++){
    	 if (error = pthread_join(*workers+i, NULL))
		printf("FUCK");
    }
    for(int i = 0; i < num_dispatcher; i++){
	      if (error = pthread_join(*dispatchers+i,NULL))
		printf("FUCK DISPATCH");
    }

    // Clean up
    free(&queue);
    delete_cache();

    return 0;
}
```