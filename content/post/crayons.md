using Crayons
using Colors



using TestImages
using Images
using Colors
using Crayons




Crayons.Crayon(rgb::RGB) = Crayon(foreground = (round(Int, 255 *rgb.r),
                                                round(Int, 255 *rgb.g),
                                                round(Int, 255 *rgb.b)))


print_with_colormap(str::String, cmapstr::String) = print_with_colormap(STDOUT, str, cmapstr)

function print_with_colormap(io::IO, str::String, cmapstr::String)
	cmap = colormap(cmapstr, strwidth(str))
	for (col, char) in zip(cmap, str)
		print(io, Crayon(col)(string(char)))
	end
end



show_image(img) = show_image(STDOUT, img)
function show_image(io::IO, img)
	c = Crayon()
	for j in 1:size(img, 2)
		for i in 1:size(img, 1)
			c = Crayon(img[i,j])
			print(io, c, "██")
		end
		print(io, "\n")
	end
end