require "octokit"

task :default => :build

class GroongaBuilder
  include Rake::DSL

  def initialize
    @top_dir = Dir.pwd
    @github_token = ENV["GITHUB_TOKEN"]
    if @github_token.nil?
      raise "must set GITHUB_TOKEN environment variable"
    end
  end

  def run
    ensure_release
    build_kytea
    build_mecab
    build_msgpack
    build_groonga
    archive_name = archive
    upload_archive(archive_name)
  end

  private
  def relative_install_prefix
    File.join("vendor", "groonga")
  end

  def absolete_install_prefix
    File.join(@top_dir, relative_install_prefix)
  end

  def relative_kytea_prefix
    File.join("vendor", "kytea")
  end

  def absolete_kytea_prefix
    File.join(@top_dir, relative_kytea_prefix)
  end

  def relative_mecab_prefix
    File.join("vendor", "mecab")
  end
  
  def absolete_mecab_prefix
    File.join(@top_dir, relative_mecab_prefix)
  end

  def mecab_config
    File.join(absolete_mecab_prefix, "bin", "mecab-config")
  end

  def groonga_version
    ENV["GROONGA_VERSION"] || "4.0.5"
  end

  def groonga_base_name
    "groonga-#{groonga_version}"
  end

  def groonga_tag_name
    "v#{groonga_version}"
  end

  def github_groonga_repository
    "groonga/groonga"
  end

  def client
    @client ||= Octokit::Client.new(:access_token => @github_token)
  end

  def find_release
    releases = client.releases(github_groonga_repository)
    releases.find do |release|
      release.tag_name == groonga_tag_name
    end
  end

  def release_exist?
    not find_release.nil?
  end

  def ensure_release
    return if release_exist?

    client.create_release(github_groonga_repository, groonga_tag_name)
  end

  def build_kytea
    kytea_version = "0.4.6"
    kytea_archive_name = "kytea-#{kytea_version}"
    sh("curl",
       "--silent",
       "--remote-name",
       "--location",
       "--fail",
       "http://www.phontron.com/kytea/download/#{kytea_archive_name}.tar.gz")
    sh("tar", "xf", "#{kytea_archive_name}.tar.gz")

    Dir.chdir(kytea_archive_name) do
      sh("./configure",
	 "--prefix=#{absolete_kytea_prefix}")
      sh("make")
      sh("make", "install")
    end
  end

  def build_mecab
    download_url = "https://mecab.googlecode.com/files"
    mecab_version = "0.996"
    mecab_archive_name = "mecab-#{mecab_version}"
    sh("curl",
       "--silent",
       "--remote-name",
       "--location",
       "--fail",
       "#{download_url}/#{mecab_archive_name}.tar.gz")
    sh("tar", "xf", "#{mecab_archive_name}.tar.gz")

    Dir.chdir(mecab_archive_name) do
      sh("./configure",
	 "--prefix=#{absolete_mecab_prefix}")
      sh("make")
      sh("make", "check")
      sh("make", "install")
    end

    ipadic_version = "2.7.0-20070801"
    ipadic_archive_name = "mecab-ipadic-#{ipadic_version}"
    sh("curl",
       "--silent",
       "--remote-name",
       "--location",
       "--fail",
       "#{download_url}/#{ipadic_archive_name}.tar.gz")
    sh("tar", "xf", "#{ipadic_archive_name}.tar.gz")
    Dir.chdir(ipadic_archive_name) do
      sh("./configure",
	 "--prefix=#{absolete_mecab_prefix}",
	 "--with-mecab-config=#{mecab_config}")
      sh("make")
      sh("make", "install")
    end
  end

  def build_msgpack
    msgpack_version = "0.5.9"
    msgpack_archive_name = "msgpack-#{msgpack_version}"
    sh("curl",
       "--silent",
       "--remote-name",
       "--location",
       "--fail",
       "https://github.com/msgpack/msgpack-c/releases/download/cpp-#{msgpack_version}/#{msgpack_archive_name}.tar.gz")
    sh("tar", "xf", "#{msgpack_archive_name}.tar.gz")

    Dir.chdir(msgpack_archive_name) do
      sh("./configure",
         "--prefix=#{absolete_install_prefix}")
      sh("make", "-j4")
      sh("make", "install")
    end
  end

  def build_groonga
    archive_name = "#{groonga_base_name}.tar.gz"
    sh("curl",
       "--silent",
       "--remote-name",
       "--location",
       "--fail",
       "http://packages.groonga.org/source/groonga/#{archive_name}")
    sh("tar", "xf", archive_name)

    Dir.chdir(groonga_base_name) do
      configure_args = []
      if ENV["DEBUG"] == "yes"
        configure_args << "--enable-debug"
      end
      sh("./configure",
         "--prefix=#{absolete_install_prefix}",
         "--disable-static",
         "--disable-document",
         "--with-message-pack=#{absolete_install_prefix}",
         *configure_args)
      sh("make", "-j4")
      sh("make", "install")
    end
  end

  def archive
    archive_name = "heroku-#{groonga_base_name}.tar.xz"
    sh("tar", "cJf", archive_name, relative_install_prefix, relative_mecab_prefix, relative_kytea_prefix)
    archive_name
  end

  def upload_archive(archive_name)
    release = find_release
    options = {
      :content_type => "application/x-xz",
    }
    client.upload_asset(release.url, archive_name, options)
  end
end

task :build do
  builder = GroongaBuilder.new
  builder.run
end
