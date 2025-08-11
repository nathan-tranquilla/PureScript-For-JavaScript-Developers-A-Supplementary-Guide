require 'fileutils'

task :run  do 
  system "mdbook serve --open"
end

task :buildprod do
  system "mdbook build"
  mv("./book/", "./docs/")
end