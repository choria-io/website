desc "Build website"
task :build_docs do
  Dir.chdir(File.dirname(__FILE__)) do
    sh("rm -rf out")
    sh("mkdir -p out/docs")
    sh("cp -r docs docs_template")
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "landing")) do
    sh("hugo -b http://%s/ -d ../out/" % ENV["CHORIA_SITE_NAME"])
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "docs")) do
    sh("hugo -b http://%s/docs/ -d ../out/docs/" % ENV["CHORIA_SITE_NAME"])
  end

  Dir.entries(File.dirname(__FILE__)).grep(/docs_\d+\.\d+\.\d+/).each do |docs_dir|
    if docs_dir =~ /docs_(\d+\.\d+\.\d+)/
      version = $1
      source_dir = File.join(File.dirname(__FILE__), docs_dir)
      out_dir = "out/docs/%s" % version

      sh("mkdir -p %s" % out_dir)

      Dir.chdir(File.join(File.dirname(__FILE__), docs_dir)) do
        sh("rm -rf ../docs_template/content")
        sh("rm -rf ../docs_template/static")
        sh("cp -r content ../docs_template")
        sh("cp -r static ../docs_template")
      end

      Dir.chdir(File.join(File.dirname(__FILE__), "docs_template")) do
        sh("hugo -b http://%s/docs/%s/ -d ../%s" % [ENV["CHORIA_SITE_NAME"], version, out_dir])
      end
    end
  end

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    rm("apple-touch-icon.png")
    rm("favicon.ico")
  end
end

desc "Build and Publish the production website"
task :publish_prod_docs do
  ENV["CHORIA_SITE_NAME"] = "choria.io"

  Rake::Task[:build_docs].invoke

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    sh("surge -d choria.io -p .")
  end
end

desc "Build and Publish the preview website"
task :publish_docs do
  ENV["CHORIA_SITE_NAME"] = "dev.choria.io"

  Rake::Task[:build_docs].invoke

  Dir.chdir(File.join(File.dirname(__FILE__), "out")) do
    sh("surge -d dev.choria.io -p .")
  end
end
