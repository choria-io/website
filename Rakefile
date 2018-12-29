desc "Build website"
task :build_docs do
  Dir.chdir(File.dirname(__FILE__)) do
    sh("rm -rf out docs_template")
    sh("mkdir -p out/docs")
    sh("cp -r docs docs_template")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "landing")) do
    sh("hugo -b %s/ -d ../out/" % ENV["CHORIA_SITE_NAME"])
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "docs")) do
    sh("hugo -b %s/docs/ -d ../out/docs/" % ENV["CHORIA_SITE_NAME"])
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "blog")) do
    sh("hugo -b %s/blog/ -d ../out/blog/" % ENV["CHORIA_SITE_NAME"])
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    rm("apple-touch-icon.png")
  end
end

desc "Build and Publish the production website"
task :publish_prod_docs do
  ENV["CHORIA_SITE_NAME"] = "https://choria.io"

  Rake::Task[:build_docs].invoke

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    sh("netlify deploy -s wonderful-curie-7b58ce -p .")
  end
end

desc "Build and Publish the preview website"
task :publish_docs do
  ENV["CHORIA_SITE_NAME"] = "https://master.choria.io"

  Rake::Task[:build_docs].invoke

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    sh("netlify deploy -s e293fe95-122a-4b0a-ae62-c74fa588ab8d -p -d .")
  end
end

desc "Build for local consumption "
task :publish_local_docs do
  ENV["CHORIA_SITE_NAME"] = "http://localhost:8080" % Dir.pwd

  Rake::Task[:build_docs].invoke

  puts "execute: ruby -run -e httpd -- out"
end
