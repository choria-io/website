desc "Build and publish the production website"
task :publish_prod_docs do
  Dir.chdir(File.join(File.dirname(__FILE__), "landing")) do
    sh("hugo -b http://choria.io")
    sh("surge -d choria.io -p public")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "docs")) do
    sh("hugo -b http://docs.choria.io")
    sh("surge -d docs.choria.io -p public")
  end
end

desc "Build and publish the preview website"
task :publish_docs do
  Dir.chdir(File.join(File.dirname(__FILE__), "landing")) do
    sh("hugo -b http://dev.choria.io")
    sh("surge -d dev.choria.io -p public")
  end
end
