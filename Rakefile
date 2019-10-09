require 'yaml'
require 'json'
require 'faraday'
require 'dotenv'

Dotenv.load

SIDENAV_INDEX_DEFAULT_TITLE = 'Overview'
SIDENAV_FILE_BLACKLIST = [
  './vendor/**/*.md',
  './src/connections/**/**.md',
  './src/*.md',
  './src/_*/**/*.md'
]

PLATFORM_API_URL = "https://platform.segmentapis.com"

namespace :nav do
  desc 'Updates _data/sidenav.yml based on the current available docs'
  task :update do
    unless ENV['FORCE_NAV_UPDATE']
      p "WARNING. This a destructive action and will break the current sidenav. Re-run with FORCE_NAV_UPDATE=1 if you are sure you want to do this"
      exit
    end

    p 'Updating _data/sidenav.yml based on current docs...'

    docs = FileList.new('./src/**/*.md').exclude(SIDENAV_FILE_BLACKLIST)
    sections = Hash.new()

    docs.map do |file_list|
      paths = file_list.to_s
                      .split('/')
                      .reject { |p| p == '.'}


      if paths.size == 0 || paths.size > 3
        # Not a valid path or some deep-nested directory structure we don't support
        p "skipping #{paths.join("/")}"
        next
      end

      root = paths[0]
      k = root


      path = nil
      path_title = nil
      path = paths[0..paths.size].join("/").gsub(".md", "")

      begin
        f = YAML.load_file("./#{path}.md")
        path_title = f["title"]
      rescue
        path_title = path.split("/").last.gsub("-", " ").capitalize
      end

      sections[k] ||= { 'section_title' => root.capitalize, 'section' => [{ 'path' => "/#{root}", 'title' => SIDENAV_INDEX_DEFAULT_TITLE}] }

      # Skip the index e.g overview page for each section since we are setting it above
      if paths.join("/") == "#{root}/index.md"
        p "skipping #{paths.join("/")}"
        next
      else
        p "found #{paths.join("/")}"
      end

      if paths.size == 3
        subsection_title = nil

        begin
          f = YAML.load_file("./#{paths[0]}/#{paths[1]}/index.md")
          subsection_title = f["title"]
        rescue
          subsection_title = paths[1].gsub("-", " ").capitalize
        end

        if !subsection = sections[k]['section'].select { |x| x['section_title'] == subsection_title }.first
          # new sub-section found
          subsection = { 'section_title' => subsection_title, 'slug' => paths[0...2].join('/'), 'section' => [] }
          sections[k]['section'] <<  subsection
        end

        # Skip the index page for the subsection
        next if path.gsub("/index", "") == paths[0...2].join("/")

        subsection['section'] << { 'path' => "/#{path}", 'title' => path_title }
      else
        sections[k]['section'] << { 'path' => "/#{path}", 'title' => path_title }
      end
    end


    main_sections = {}
    legal_sections = {}
    api_sections = {}
    partners_sections = {}

    sections.each do |k, v|
      if k == 'legal'
        legal_sections[k] = v
      elsif k == 'api'
        api_sections[k] = v
      elsif k == 'partners'
        partners_sections[k] = v
      else
        main_sections[k] = v
      end
    end

    main_nav = { 'sections' => main_sections.values }
    legal_nav = { 'sections' => legal_sections.values }
    api_nav = { 'sections' => api_sections.values }
    partners_nav = { 'sections' => partners_sections.values }

    # Main sidenav
    File.open("./src/_data/sidenav/main.yml","w") do |file|
      file.write main_nav.to_yaml({ indention: 4, separator: '' })
    end

    # Legal sidenav
    File.open("./src/_data/sidenav/legal.yml","w") do |file|
      file.write legal_nav.to_yaml({ indention: 4, separator: '' })
    end

    # API sidenav
    File.open("./src/_data/sidenav/api.yml","w") do |file|
      file.write api_nav.to_yaml({ indention: 4, separator: '' })
    end

    # Partners sidenav
    File.open("./src/_data/sidenav/partners.yml","w") do |file|
      file.write partners_nav.to_yaml({ indention: 4, separator: '' })
    end
  end
end

namespace :catalog do
  desc 'Updates the catalog data files based on the current data available in the Platform API'
  task :update do
    if !ENV['PLATFORM_API_TOKEN']
      p "No API Key found. Skipping catalog updates from Platform API..."
      exit
    end

    p "Saving catalogs from Platform API..."
    Rake::Task["catalog:update_destinations"].invoke
    Rake::Task["catalog:update_sources"].invoke
    p "Done."
  end

  desc 'Updates the destination catalog data file based on the current data available in the Platform API'
  task :update_destinations do
    destinations = []
    begin
      resp = {}
      next_page_token = nil
      loop do
        resp = get_catalog(url: "#{PLATFORM_API_URL}/v1beta/catalog/destinations", page_token: next_page_token)
        next_page_token = resp.delete("next_page_token")
        destinations = (destinations << resp['destinations']).flatten

        break if next_page_token == ""
      end

    rescue Exception => e
      abort e
    end

    destinations.sort_by! { |d| d['name'] }

    File.open("./src/_data/catalog/destinations.yml","w") do |file|
      file.write "# AUTOGENERATED FROM PLATFORM API. DO NOT EDIT\n"
      file.write(({"destinations" => destinations}).to_yaml({ indention: 4, separator: '' }))
    end

    p "Finished Destinations."
  end

  desc 'Updates the source catalog data file based on the current data available in the Platform API'
  task :update_sources do
    sources = []
    begin
      resp = {}
      next_page_token = nil
      loop do
        resp = get_catalog(url: "#{PLATFORM_API_URL}/v1beta/catalog/sources", page_token: next_page_token)
        next_page_token = resp.delete("next_page_token")
        sources = (sources << resp['sources']).flatten

        break if next_page_token == ""
      end

    rescue Exception => e
      abort e
    end

    sources.sort_by! { |d| d['name'] }

    File.open("./src/_data/catalog/sources.yml","w") do |file|
      file.write "# AUTOGENERATED FROM PLATFORM API. DO NOT EDIT\n"
      file.write(({"sources" => sources}).to_yaml({ indention: 4, separator: '' }))
    end

    p "Finished Sources."
  end
end


def get_catalog(url:, page_token: "")
  resp = Faraday.get(url) do |req|
    req.params['page_token'] = page_token
    req.params['page_size'] = 100
    req.headers['Content-Type'] = 'application/json'
    req.headers['Authorization'] = "Bearer #{ENV["PLATFORM_API_TOKEN"]}"
  end


  JSON.parse(resp.body)
end
