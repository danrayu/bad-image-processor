#include <stdio.h>
#include <png++/png.hpp>
#include <stdexcept>

int width, height;
png_byte color_type;
png_byte bit_depth;
png_bytep *row_pointers = NULL;

void read_png_file(char *filename) {
  FILE *fp = fopen(filename, "rb");
  png_structp png = png_create_read_struct(PNG_LIBPNG_VER_STRING, NULL, NULL, NULL);
  if(!png) throw std::logic_error("Failed PNG read struct creation");

  png_infop info = png_create_info_struct(png);
  if(!info) throw std::logic_error("Failed creating info struct");

  if(setjmp(png_jmpbuf(png))) throw std::logic_error("Failed to set jump");

  png_init_io(png, fp);

  png_read_info(png, info);

  width      = png_get_image_width(png, info);
  height     = png_get_image_height(png, info);
  color_type = png_get_color_type(png, info);
  bit_depth  = png_get_bit_depth(png, info);

  // Read any color_type into 8bit depth, RGBA format.
  // See http://www.libpng.org/pub/png/libpng-manual.txt

  if(bit_depth == 16)
	png_set_strip_16(png);

  if(color_type == PNG_COLOR_TYPE_PALETTE)
	png_set_palette_to_rgb(png);

  // PNG_COLOR_TYPE_GRAY_ALPHA is always 8 or 16bit depth.
  if(color_type == PNG_COLOR_TYPE_GRAY && bit_depth < 8)
	png_set_expand_gray_1_2_4_to_8(png);

  if(png_get_valid(png, info, PNG_INFO_tRNS))
	png_set_tRNS_to_alpha(png);

  // These color_type don't have an alpha channel then fill it with 0xff.
  if(color_type == PNG_COLOR_TYPE_RGB ||
	 color_type == PNG_COLOR_TYPE_GRAY ||
	 color_type == PNG_COLOR_TYPE_PALETTE)
	png_set_filler(png, 0xFF, PNG_FILLER_AFTER);

  if(color_type == PNG_COLOR_TYPE_GRAY ||
	 color_type == PNG_COLOR_TYPE_GRAY_ALPHA)
	png_set_gray_to_rgb(png);

  png_read_update_info(png, info);

  if (row_pointers) abort();

  row_pointers = (png_bytep*)malloc(sizeof(png_bytep) * height);
  for(int y = 0; y < height; y++) {
	row_pointers[y] = (png_byte*)malloc(png_get_rowbytes(png,info));
  }

  png_read_image(png, row_pointers);
  fclose(fp);

  png_destroy_read_struct(&png, &info, NULL);
}

void write_png_file(char *filename) {
  int y;

  FILE *fp = fopen(filename, "wb");
  if(!fp) throw std::logic_error("Failed to open file");

  png_structp png = png_create_write_struct(PNG_LIBPNG_VER_STRING, NULL, NULL, NULL);
  if (!png) throw std::logic_error("Failed to create write struct");

  png_infop info = png_create_info_struct(png);
  if (!info) throw std::logic_error("Failed to create info struct");

  if (setjmp(png_jmpbuf(png))) abort();

  png_init_io(png, fp);

  // Output is 8bit depth, RGBA format.
  png_set_IHDR(
	png,
	info,
	width, height,
	8,
	PNG_COLOR_TYPE_RGBA,
	PNG_INTERLACE_NONE,
	PNG_COMPRESSION_TYPE_DEFAULT,
	PNG_FILTER_TYPE_DEFAULT
  );
  png_write_info(png, info);

  // To remove the alpha channel for PNG_COLOR_TYPE_RGB format,
  // Use png_set_filler().
  //png_set_filler(png, 0, PNG_FILLER_AFTER);

  if (!row_pointers)  throw std::logic_error("Row pointers not valid");;

  png_write_image(png, row_pointers);
  png_write_end(png, NULL);

  for(int y = 0; y < height; y++) {
	free(row_pointers[y]);
  }
  free(row_pointers);

  fclose(fp);

  png_destroy_write_struct(&png, &info);
}

void lowPassFilter(png_bytep * rows, int strength) {
	for (int i = 0; i < strength; i++) {
		for(int y = 0; y < height; y++) {
			png_bytep row = rows[y];
			for(int x = 0; x < width; x++) {
				int r = 0;
				int g = 0;
				int b = 0;
				png_bytep px = &(row[x * 4]);
				for (int d_x = -1; d_x < 2; d_x++) {
					for (int d_y = -1; d_y < 2; d_y++) {
						if (x + d_x >= 0 && y + d_y >= 0 && y + d_y <=
							height - 1 && x + d_x <= width - 1) 
						{
							r += (&((row_pointers[y + d_y])[(x + d_x) * 4]))[0];
							b += (&((row_pointers[y + d_y])[(x + d_x) * 4]))[1];
							g += (&((row_pointers[y + d_y])[(x + d_x) * 4]))[2];
						} else {
							r += px[0];
							b += px[1];
							g += px[2];
						}
					}
				}
				r /= 9;
				b /= 9;
				g /= 9;

				px[0] = r;
				px[1] = b;
				px[2] = g;
			}
		}
	}
}

void invertBrightness(png_bytep* rows) {
	for(int y = 0; y < height; y++) {
		png_bytep row = rows[y];
		for(int x = 0; x < width; x++) {
			png_bytep px = &(row[x * 4]);
			px[0] = 255 - px[0];
			px[1] = 255 - px[1];
			px[2] = 255 - px[2];
		}
	}
}

void simpleScale(png_bytep* rows, float intensity) {
	for(int y = 0; y < height; y++) {
		png_bytep row = rows[y];
		for(int x = 0; x < width; x++) {
			png_bytep px = &(row[x * 4]);
			px[0] *= intensity;
			px[1] *= intensity;
			px[2] *= intensity;
		}
	}
}

void addImages(png_bytep* base, png_bytep* addition) {
	for(int y = 0; y < height; y++) {
		png_bytep rowB = base[y];
		png_bytep rowA = addition[y];
		for(int x = 0; x < width; x++) {
			png_bytep pxA = &(rowA[x * 4]);
			png_bytep pxB = &(rowB[x * 4]);
			pxB[0] += pxA[0];
			pxB[1] += pxA[1];
			pxB[2] += pxA[2];
		}
	}
}

void substractScaledImage(png_bytep* base, png_bytep* addition, float strength) {
	for(int y = 0; y < height; y++) {
		png_bytep rowB = base[y];
		png_bytep rowA = addition[y];
		for(int x = 0; x < width; x++) {
			png_bytep pxA = &(rowA[x * 4]);
			png_bytep pxB = &(rowB[x * 4]);
			pxB[0] = pxB[0] - pxA[0];
			pxB[1] = pxB[1] - pxA[1];
			pxB[2] = pxB[2] - pxA[2];
		}
	}
}

png_bytep * copyRowPointers(png_bytep* orginal) {
	png_bytep * copiedRowPointers = (png_bytep*) malloc(height * sizeof(png_bytep));
	for (int y = 0; y < height; y++) {
		png_bytep copied_row = (png_bytep) malloc(width * sizeof(png_byte) * 4);
		memcpy(copied_row, orginal[y], width * 4);
		copiedRowPointers[y] = copied_row;
	}
	return copiedRowPointers;
}

void highBoostSharpen(png_bytep* rows, int strength) {
	png_bytep * copiedRowPointers = copyRowPointers(rows);
	// lowPassFilter(rows, 70); remove for interesting results
	lowPassFilter(copiedRowPointers, 4);
	for(int y = 0; y < height; y++) {
		png_bytep rowB = rows[y];
		png_bytep rowA = copiedRowPointers[y];
		for(int x = 0; x < width; x++) {
			png_bytep pxA = &(rowA[x * 4]);
			png_bytep pxB = &(rowB[x * 4]);
			int result_r = pxB[0] + strength * (pxB[0] - pxA[0]);
			pxB[0] = result_r;
			if (result_r < 0) {
				pxB[0] = 0;
			}
			if (result_r > 255) {
				pxB[0] = 255;
			}
			int result_b = pxB[1] + strength * (pxB[1] - pxA[1]);
			pxB[1] = result_b;
			if (result_b < 0) {
				pxB[1] = 0;
			}
			if (result_b > 255) {
				pxB[1] = 255;
			}
			int result_g = pxB[2] + strength * (pxB[2] - pxA[2]);
			pxB[2] = result_g;
			if (result_g < 0) {
				pxB[2] = 0;
			}
			if (result_g > 255) {
				pxB[2] = 255;
			}
		}
	}
	free(copiedRowPointers);
}

void process_png_file(int strength) {
	for (int i = 0; i < strength; i ++) {
		highBoostSharpen(row_pointers, 2); // set to 400 for interesting results
	}
}

int main(int argc, char *argv[]) {
  if(argc != 3)  throw std::logic_error("Argc not 3");

  read_png_file(argv[1]);
  process_png_file(1);
  write_png_file(argv[2]);

  return 0;
}
