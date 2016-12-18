desc "Build and publish the production website"
task :publish_prod_docs do
  Dir.chdir(File.dirname(__FILE__)) do
    sh("rm -rf out")
    sh("mkdir -p out/docs")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "landing")) do
    sh("hugo -b http://choria.io/ -d ../out/")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "docs")) do
    sh("hugo -b http://choria.io/docs/ -d ../out/docs/")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    rm("apple-touch-icon.png")
    rm("favicon.ico")
    sh("surge -d choria.io -p .")
  end
end

desc "Build and publish the preview website"
task :publish_docs do
  Dir.chdir(File.dirname(__FILE__)) do
    sh("rm -rf out")
    sh("mkdir -p out/docs")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "landing")) do
    sh("hugo -b http://dev.choria.io/ -d ../out/")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "docs")) do
    sh("hugo -b http://dev.choria.io/docs/ -d ../out/docs/")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    sh("surge -d dev.choria.io -p .")
  end
end
