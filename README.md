# Longhair
## Fast Cauchy Reed-Solomon Erasure Codes in C

Longhair is a simple, portable library for erasure codes.  From given data it generates
redundant data that can be used to recover the originals.  It is extremely fast, though
not as fast as the [CM256 library](https://github.com/catid/cm256).

The original data should be split up into equally-sized chunks.  If one of these chunks
is erased, the redundant data can fill in the gap through decoding.

The erasure code is parameterized by three values (`k`, `m`, `bytes`).  These are:

+ The number of blocks of original data (`k`), which must be less than 256.
+ The number of blocks of redundant data (`m`), which must be no more than `256 - k`.
+ And the number of bytes per block (`bytes`), which must be a multiple of 8 bytes.

These erasure codes are not patent-encumbered and the software is provided royalty-free.


##### Building: Quick Setup

The [longhair-mobile](https://github.com/catid/longhair/tree/master/longhair-mobile)
directory contains an easy-to-import set of C code that also
builds properly for mobile devices.  In a pinch you can use this code for
desktops too.


## Usage

Documentation is provided in the header file [cauchy_256.h](https://github.com/catid/longhair/raw/master/cauchy_256.h).

When your application starts up it should call `cauchy_init()` to verify that the library is linked properly:

~~~
	#include "cauchy_256.h"

	if (cauchy_init()) {
		// Wrong static library
		exit(1);
	}
~~~

To generate redundancy, use the `cauchy_256_encode` function:

~~~
	int data_len = ...;

	int k = 32; // Choose a number of pieces to split the data into.
	int m = 12; // Choose a number of redundant pieces to generate.
	int bytes = 1000; // Chose a number of bytes per chunk, a multiple of 8 bytes.

	assert(bytes % 8 == 0);
	assert(data_len == k * bytes);

	char *recovery_blocks = new char[m * bytes];
	char *data_ptrs[32];

	// Point data_ptrs to data here

	// Encode the data using this library
	if (cauchy_256_encode(k, m, data_ptrs, recovery_blocks, bytes)) {
		// Input was invalid
		return false;
	}

	// For each recovery block,
	for (int ii = 0; ii < m; ++ii) {
		char *block = recovery_blocks + ii * bytes;
		unsigned char row = k + ii; // Transmit this with block (just one byte)

		// Transmit or store block bytes and row
	}

	delete []recovery_blocks;
~~~

To recover the original data, use the `cauchy_256_decode` function:

~~~
	// Use the same settings as the encoder
	int k = 32;
	int m = 12;
	int bytes = 1000;

	// Allocate block info array
	// There should be exactly k blocks
	Block *block_info = new Block[k];

	// Fill block_info here with data and rows from the encoder
	// Rows under k are original data, and the rest are redundant data

	// Attempt decoding
	if (cauchy_256_decode(k, m, block_info, bytes)) {
		// Decoding should never fail - indicates input is invalid
		assert(k + m <= 256);
		assert(block_info != 0);
		assert(bytes % 8 == 0);
		return false;
	}

	// Now the block_info elements that used to have redundant data are
	// corrected in-place and contain the original data.
~~~

The example above is just one way to use the `cauchy_256_decode` function.

This API was designed to be flexible enough for UDP/IP-based file transfer where
the blocks arrive out of order.  For packet erasure correction the following code
is more applicable:

~~~
	// Use the same settings as the encoder
	int k = 32;
	int m = 12;
	int bytes = 1000;

	// Allocate enough space for the original data
	char *block_data = new char[bytes * k];

	// Allocate block info array
	Block *block_info = new Block[k];

	int original_count = 0, recovery_count = 0;

	// This function will be called by onData() with original data after recovery
	static void processData(int row, char *data, int data_bytes) {
		// Handle original data here only
	}

	// Call this function with each block received, either original or recovery
	// Returns true on complete
	bool onData(unsigned char row, char *new_data) {
		int insertion_point;

		// If it is original data,
		if (row < k) {
			// Process the original data immediately - Do not wait for it all to arrive!
			processData(row, new_data, bytes);

			// Copy to the end of the original block data
			insertion_point = original_count++;
		} else {
			// Copy to the front of the recovery block data
			insertion_point = k - ++recovery_count;
		}

		// Copy data into place
		char *dest = block_data + insertion_point * bytes;
		memcpy(dest, new_data, bytes);

		// NOTE: It may be possible to avoid copying depending on if
		// you can hang onto the provided data buffer.

		// Fill in the block array entry
		Block *block = block_info + insertion_point;
		block->data = dest;
		block->row = row;

		// If recovery is not possible yet,
		if (original_count + recovery_count < k) {
			return false;
		}

		// Attempt decoding
		if (cauchy_256_decode(k, m, block_info, bytes)) {
			// Decoding should never fail - indicates input is invalid
			assert(k + m <= 256);
			assert(block_info != 0);
			assert(bytes % 8 == 0);
			return false;
		}

		// For each recovered block,
		block = block_info + k - recovery_count;
		for (int ii = 0; ii < recovery_count; ++ii, ++block) {
			// Process the recovered data
			processData(block->row, block->data, bytes);
		}

		return true;
	}
~~~


## Benchmarks

This is running on my pretty fast desktop.  Try it out on your target device and see how it does!

~~~
Using 1296 bytes per block (ie. packet/chunk size); must be a multiple of 8 bytes

Encoded k=29 data blocks with m=1 recovery blocks in 1 usec : 37584 MB/s
+ Decoded 1 erasures in 2 usec : 18792 MB/s
Encoded k=29 data blocks with m=2 recovery blocks in 9 usec : 4176 MB/s
+ Decoded 2 erasures in 12 usec : 3132 MB/s
Encoded k=29 data blocks with m=3 recovery blocks in 19 usec : 1978 MB/s
+ Decoded 3 erasures in 22 usec : 1708 MB/s
Encoded k=29 data blocks with m=4 recovery blocks in 31 usec : 1212 MB/s
+ Decoded 4 erasures in 35 usec : 1073 MB/s
Encoded k=29 data blocks with m=5 recovery blocks in 22 usec : 1708 MB/s
+ Decoded 5 erasures in 30 usec : 1252 MB/s
Encoded k=29 data blocks with m=6 recovery blocks in 24 usec : 1566 MB/s
+ Decoded 6 erasures in 34 usec : 1105 MB/s
Encoded k=29 data blocks with m=7 recovery blocks in 28 usec : 1342 MB/s
+ Decoded 7 erasures in 39 usec : 963 MB/s
Encoded k=29 data blocks with m=8 recovery blocks in 30 usec : 1252 MB/s
+ Decoded 8 erasures in 43 usec : 874 MB/s
Encoded k=29 data blocks with m=9 recovery blocks in 33 usec : 1138 MB/s
+ Decoded 9 erasures in 53 usec : 709 MB/s
Encoded k=29 data blocks with m=10 recovery blocks in 36 usec : 1044 MB/s
+ Decoded 10 erasures in 57 usec : 659 MB/s
Encoded k=29 data blocks with m=11 recovery blocks in 39 usec : 963 MB/s
+ Decoded 11 erasures in 63 usec : 596 MB/s
Encoded k=29 data blocks with m=12 recovery blocks in 41 usec : 916 MB/s
+ Decoded 12 erasures in 71 usec : 529 MB/s
Encoded k=29 data blocks with m=13 recovery blocks in 51 usec : 736 MB/s
+ Decoded 13 erasures in 82 usec : 458 MB/s
Encoded k=29 data blocks with m=14 recovery blocks in 48 usec : 783 MB/s
+ Decoded 14 erasures in 87 usec : 432 MB/s
~~~

Note that the codec is specialized for the `m = 1` case and runs very quickly.
Due to a happy coincidence the first recovery block is always just an XOR of
all the original data, so you can use this codec instead of doing that manually.

For erasure codes the important statistic to watch is the encoder speed, because
it determines whether or not the transmitter can afford to send the extra data.
Usually the decoder is not going to be using every redundant data block, so
the benchmark is actually worst-case figures for the decoder.


## Comparisons with Alternatives

This library implements Cauchy Reed-Solomon (CRS) codes as introduced by
[Jerasure](https://github.com/tsuraan/Jerasure).  This library improves on
existing CRS codes in a number of novel ways to greatly increase throughput.
For full details on what has been improved, refer to the source code comments.

Jerasure performance for the packet case can be improved a lot.  Jerasure has
the best performance when the value of "w" is tuned to be as small as possible,
whereas it is fixed at w = 8 in Longhair.  For a favorable benchmark, set w = 6
and k = 29, and recovery block count m = 1-5:

##### Jerasure unit tester on iMac (2.7 GHz Core i5-2500S Sandy Bridge, June 2011):

With "smart scheduling": (Does not help for streaming scenario)

~~~
Using 1488 bytes per block (ie. packet/chunk size); must be a multiple of 8 bytes

Encoded k=29 data blocks with m=1 recovery blocks in 20 usec : 2157.6 MB/s
Encoded k=29 data blocks with m=2 recovery blocks in 91 usec : 474.198 MB/s
Encoded k=29 data blocks with m=3 recovery blocks in 167 usec : 258.395 MB/s
Encoded k=29 data blocks with m=4 recovery blocks in 240 usec : 179.8 MB/s
Encoded k=29 data blocks with m=5 recovery blocks in 322 usec : 134.012 MB/s
~~~

Without "smart scheduling" and with "improved coding matrix": (Setup time hurts performance)

~~~
Encoded k=29 data blocks with m=1 recovery blocks in 9 usec : 4794.67 MB/s
Encoded k=29 data blocks with m=2 recovery blocks in 53 usec : 814.189 MB/s
Encoded k=29 data blocks with m=3 recovery blocks in 98 usec : 440.327 MB/s
Encoded k=29 data blocks with m=4 recovery blocks in 152 usec : 283.895 MB/s
Encoded k=29 data blocks with m=5 recovery blocks in 187 usec : 230.759 MB/s
~~~

Without "improved coding matrix" and without "smart scheduling": (BEST PERFORMANCE)

~~~
Encoded k=29 data blocks with m=1 recovery blocks in 30 usec : 1438.4 MB/s
Encoded k=29 data blocks with m=2 recovery blocks in 52 usec : 829.846 MB/s
Encoded k=29 data blocks with m=3 recovery blocks in 77 usec : 560.416 MB/s
Encoded k=29 data blocks with m=4 recovery blocks in 104 usec : 414.923 MB/s
Encoded k=29 data blocks with m=5 recovery blocks in 127 usec : 339.78 MB/s
Encoded k=29 data blocks with m=6 recovery blocks in 153 usec : 282.039 MB/s
~~~

Note Jerasure is roughly 3x slower than this library for the streaming use case.


##### Rateless codes

There is another alternative way to do erasure codes efficiently called rateless
codes, where there is no need to specify a value for `m` and you can generate as
many redundant blocks as needed.  Rateless codes tend to be more efficient for
high-speed file transfer.  For more information on these types of erasure codes
see the [Wirehair](https://github.com/catid/wirehair) library.


## Encoder speed discussion

I found that the encoder speed is directly related to the number of error
correction blocks and doesn't depend on the amount of data to protect:

![alt text](https://github.com/catid/longhair/raw/master/docs/EncoderSpeed.png "Speed of Encoder for k, m")

The rows are values of k (amount of data) and the columns are values of m (number of recovery blocks added).

Darker is better.  The main point of this plot is to just show that k doesn't
factor into the performance of the code.


#### Credits

Software by Christopher A. Taylor mrcatid@gmail.com

Please reach out if you need support or would like to collaborate on a project.
