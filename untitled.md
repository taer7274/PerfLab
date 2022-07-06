# Performance Lab
Our goal here is to improve the image filtering program by a factor of about 25.

We know we need to implement the following improvements:

- Loop order
- Strength Reduction
- Code Motion
- Loop Unrolling
- GCC Optimization Level

We will do these things one by one, as we see them applicable to the code. 

We're given in the instructions that there is a central function that does a lot of the work: 

    double
    applyFilter(struct Filter *filter, cs1300bmp *input, cs1300bmp *output)
    {

      long long cycStart, cycStop;

      cycStart = rdtscll();

      output -> width = input -> width;
      output -> height = input -> height;


      for(int col = 1; col < (input -> width) - 1; col = col + 1) {
        for(int row = 1; row < (input -> height) - 1 ; row = row + 1) {
          for(int plane = 0; plane < 3; plane++) {

      output -> color[plane][row][col] = 0;

      for (int j = 0; j < filter -> getSize(); j++) {
        for (int i = 0; i < filter -> getSize(); i++) {	
          output -> color[plane][row][col]
            = output -> color[plane][row][col]
            + (input -> color[plane][row + i - 1][col + j - 1] 
         * filter -> get(i, j) );
        }
      }

      output -> color[plane][row][col] = 	
        output -> color[plane][row][col] / filter -> getDivisor();

      if ( output -> color[plane][row][col]  < 0 ) {
        output -> color[plane][row][col] = 0;
      }

      if ( output -> color[plane][row][col]  > 255 ) { 
        output -> color[plane][row][col] = 255;
      }
          }
        }
      }

      cycStop = rdtscll();
      double diff = cycStop - cycStart;
      double diffPerPixel = diff / (output -> width * output -> height);
      fprintf(stderr, "Took %f cycles to process, or %f cycles per pixel\n",
        diff, diff / (output -> width * output -> height));
      return diffPerPixel;
    }
    
    
This seems like the best place to start, since we can clearly see that these nested loops are going to impact the time heavily. 

One of the first things we notice is that the results are not accumulated in a temporary variable, and that there are memory references within the loop. Let's reduce this in our main function loop (this is an example of data movement)

## Data Movement

Let's change the memory references in our loops and see what difference this makes on our run time. 

        double
        applyFilter(struct Filter *filter, cs1300bmp *input, cs1300bmp *output)
        {

          long long cycStart, cycStop;

          cycStart = rdtscll();

          output -> width = input -> width;
          output -> height = input -> height;
            int limit_w = input -> width;
            int limit_h = input -> height;
            int filter_sz = filter -> getSize();

          /* Remove memory access in loop by using temporary variables */
          /*for(int col = 1; col < (input -> width) - 1; col = col + 1) {*/
            for(int col = 1; col < limit_w - 1; col = col + 1) {
            /*for(int row = 1; row < (input -> height) - 1 ; row = row + 1) {*/
            for(int row = 1; row < limit_h - 1 ; row = row + 1) {    
              for(int plane = 0; plane < 3; plane++) {

            output -> color[plane][row][col] = 0;

            /* Reduce procedure calls in the loop */
            /*for (int j = 0; j < filter -> getSize(); j++) {*/
              for (int j = 0; j < filter_sz; j++) {
              /*for (int i = 0; i < filter -> getSize(); i++) {	*/
                  for (int i = 0; i < filter_sz; i++) {	
                    output -> color[plane][row][col]
                      = output -> color[plane][row][col]
                      + (input -> color[plane][row + i - 1][col + j - 1] 
                     * filter -> get(i, j) );
                  }
            }

            output -> color[plane][row][col] = 	
              output -> color[plane][row][col] / filter -> getDivisor();

            if ( output -> color[plane][row][col]  < 0 ) {
              output -> color[plane][row][col] = 0;
            }

            if ( output -> color[plane][row][col]  > 255 ) { 
              output -> color[plane][row][col] = 255;
            }
              }
            }
          }

          cycStop = rdtscll();
          double diff = cycStop - cycStart;
          double diffPerPixel = diff / (output -> width * output -> height);
          fprintf(stderr, "Took %f cycles to process, or %f cycles per pixel\n",
              diff, diff / (output -> width * output -> height));
          return diffPerPixel;
        }

Before we reduce the proceduce calls and memory references, our median CPE for the `boats.bmp` file is 1792. 

After our changes, our median CPE is 1655. This is not a significant change, but still an improvement. 

We notice in our editing that we can reduce a memory access for filter -> getSize(); by assigning the value of 3 to the limit. This provides a change from 924 - 901. Not a giant change, but still helpful.

We can also try to change the size of some of loop indexes to short integers. It did not provide a noticeable difference in run time, but we'll keep it anyways.

Let's keep going with our data movement and see whether we can implement any further improvements.

## Loop Ordering

We notice that the ordering of the loops is suboptimal. 

We want to follow the ordering that the memory will be accessed - plane, row, then column, not column, row, plane. 

This provides a significant improvement in our performance; we now see a median CPE of 924. This is by far one of the most significant improvements we've made to the function.



We know generally that loop unrolling is a great way to improve performance; let's give that a shot and see if we can get any noticeable improvements. 

## Loop Unrolling

Generally, loop unrolling works because it increases the number of calculations done per iteration. Additionally, it reduces the number of calculations done to support the loop, like conditional checks. 

We can unroll a loop by any factor of k, yielding k x 1 loop unrolling. We se the upper limit to be n - k + 1 and within the loop apply the combining operation to elements i through i + k - 1. Loop index i is incremented by k in each iteration. The maximum array index i + k - 1 sill always be les then n. We include the second loop to step through the final few elements of the vector one at a time. The body of the loop will be executed between 0 and k - 1 times. 

This is an example of loop unrolling from the book: 

    /* Accumulate results in local variable */
    
    void combine4(vec_ptr v, data_t *dest)
    {
        long i; 
        long length = vec_length(v);
        data_t *data = get_vec_start(v);
        data_t acc = IDENT;
        
        for (i = 0; i < length; i++){
          acc = acc OP data[i];
        }
        *dest = acc;
      }

    /* 2 x 1 loop unrolling */
    
    void combine5(vec_ptr v, data_t *dest)
    {
        long i; 
        long length = vec_length(v);
        long limit = length - 1;
        data_t *data = get_vec_start(v);
        data_t acc = IDENT;
        
        /* Combine 2 elements at a time */
        for (i = 0; i < limit; i +=2){
          acc = (ac OP data[i]) OP data[i+1];
        }
        
        /* Finish any remaining elements */
        for (; i < length; i++){
          acc = acc OP data[i];
        }
        *dest = acc;
     }
 
 
