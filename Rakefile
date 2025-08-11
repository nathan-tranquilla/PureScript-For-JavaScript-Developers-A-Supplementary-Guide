require 'fileutils'

task :run  do 
  system "mdbook serve --open"
end

task :clean do 
  rm_rf "docs"
  rm_rf "book"
end

task :buildprod => [:clean] do
  system "mdbook build"
  mv("./book/", "./docs/")
end