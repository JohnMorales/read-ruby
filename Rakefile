require 'rubygems'
require 'bundler/setup'
require 'nokogiri'
require 'rake/clean'
require 'yaml'
require 'coderay'
require 'mechanize'
# Directory containing the source XML files
SRC_DIR = 'src'

# Directory containing the source XML files for the reference section
SRC_REF_DIR = SRC_DIR + '/ref'

# Directory containing the files that the web site consists of
OUT_DIR = 'out'

# Directory containing the reference appendices for the web site
REF_DIR = OUT_DIR + '/ref'

# Directory containing the immediate results of the XSLT
BUILD_DIR = 'build'

# Directory containing the immediate results of the XSLT for reference pages
BUILD_REF_DIR = BUILD_DIR + '/ref'
 
# XSL stylesheet for transforming SRC_XML to HTML
HTML_XSL = "xsl/html5.xsl"

# XSL stylesheet for transforming SRC_XML to PDF
PDF_XSL = "xsl/pdf.xsl"

# The DocBook sources (keywords.xml is currently unused)
SRC_XML = FileList["#{SRC_DIR}/*.xml", "#{SRC_REF_DIR}/*.xml"].exclude('src/keywords.xml')

# The root file of the XML source: XIncludes SRC_DIR
BOOK_XML = 'book.xml'

# Files created by xsltproc(1) when given BOOK_XML
BUILD_HTML = SRC_XML.map{|f| f.ext('html').sub(/^src/, BUILD_DIR)}

# The BUILD_XML files destined for OUT_DIR
HTML = BUILD_HTML.map{|f| f.sub(/^#{BUILD_DIR}/, OUT_DIR)}

# The file in BUILD_DIR that should be copied to TOC_OUT. This is special-cased
# because there is no corresponding source file
TOC_BUILD = File.join(BUILD_DIR, 'toc.html')

# The destination of TOC_BUILD
TOC_OUT = TOC_BUILD.sub(/^#{BUILD_DIR}/, OUT_DIR)

# Directory containing files that should be copied to OUT_DIR
PRISTINE_DIR = 'www'

# The contents of PRISTINE_DIR
PRISTINE_FILES = FileList["#{PRISTINE_DIR}/{*,.[a-z]*}"]

# Directory containing EX_RB and EX_HTML
EX_DIR = 'examples'

# Ruby code examples
EX_RB = FileList[EX_DIR + '/*.rb']

# Highlighted copies of EX_RB
EX_HTML = EX_RB.map {|f| f.ext 'html'}

# Format string applied to EX_HTML before being embedded in HTML
CODERAY = '<a href=/examples/%s><pre class=ruby><code>%s</code></pre></a>'

# RelaxNG schema for DocBook 5 validation
# (To validate locally, set the 'docbook' symlink to point to the system
# DocBook schema/stylesheet directory. On Debian and its derivatives this is
# /usr/share/xml/docbook/)
DOCBOOK_RNG = 'docbook.rng'

# Globs that should not be rsync'd
RSYNC_EXCLUDE = %w{*examples/*html *examples/.*}

# Null device (defined by default on 1.9.3)
IO::NULL = '/dev/null' unless IO.const_defined?(:NULL)

# Root URL
URL = 'http://ruby.runpaint.org/'

# PDF version 
PDF = File.join(OUT_DIR, 'read-ruby.pdf')

# HTML used to make PDF
PDF_HTML = "#{BUILD_DIR}/single.html"

# Create OUT_DIR, EX_DIR, and BUILD_DIR as needed.
[OUT_DIR, REF_DIR, BUILD_DIR, BUILD_REF_DIR].each do |dir|
  task :html => dir
  directory dir
end

# By-products of build process
CLEAN.exclude(OUT_DIR).include(*EX_HTML, BUILD_DIR)

# All products of build process
CLOBBER.include(*EX_HTML, OUT_DIR)

GIT_DATE, GIT_HASH = `git log -n 1 --format="%ci%n%H"`.lines.map(&:chomp)

XSLTPROC = ->(xsl) do
  datetime = DateTime.parse(GIT_DATE).iso8601
  params = {
    'stringparam out_dir' => BUILD_DIR,
    'stringparam git_hash' => GIT_HASH,
    'stringparam git_date' => GIT_DATE,
    'stringparam git_datetime' => datetime,
  }.map{|n,v| "--#{n} '#{v}'"}.join(?\ )
  sh "xsltproc #{params} --xinclude #{xsl} #{BOOK_XML} >#{IO::NULL}"
end

task :html => EX_HTML

# To generate a file in HTML we need to XSLT BOOK_XML. However, the XSLT
# re-generates all of HTML; not just a single file. This is problematic because
# it clobbers the mtimes that Rake needs to operate correctly. So, instead, we
# put the results of the XSLT into BUILD_DIR, then only copy the product we are
# trying to create into OUT_DIR. This also allows us to only perform the XSLT
# once per run, as we cache the results in BUILD_DIR. This may have updated the
# ToC as well, so we copy that, too. Lastly, we need to highlight the examples
# contained in the newly-generated HTML file.


sh "mkdir -p out/ref"

HTML.zip(SRC_XML).each_with_index do |(html, src), i|
  desc "XSLT #{src} to #{html}"
  XSLTPROC[HTML_XSL] if uptodate?(src, [BUILD_HTML[i]])
  cp BUILD_HTML[i], html
  cp TOC_BUILD, TOC_OUT unless uptodate?(TOC_OUT, SRC_XML << TOC_BUILD) 
end

# Minify the HTML files in OUT_DIR, plus TOC_OUT. Then produce a gzipped
# version of the minified HTML.
(HTML << TOC_OUT).map {|f| f.ext('html.gz')}.zip(HTML).each do |min, html|
  task :html => min
  desc "Minify and compress #{html}"
  rule min => html do
    #open("#{html}.min", ?w){|f| f << HTML5.minify(File.read html)}
    #mv "#{html}.min", html
    #sh "gzip -cn #{html} >#{min}"
  end
end

# Copy over files in PRISTINE_DIR that aren't already in OUT_DIR
PRISTINE_FILES.each do |f|
  task :html => (out = f.sub(/^#{PRISTINE_DIR}/, 'out'))
  rule out => f do 
    File.symlink?(f) ? ln_s("../#{f}", out) : cp(f, out)
  end
end

# Create highlighted copies of each EX_RB file
EX_HTML.zip(EX_RB).each do |html, rb|
  desc "Highlight #{rb} with CodeRay"
  rule html => rb do
    pre = Nokogiri::HTML(CodeRay.highlight_file rb).at('pre')
    open(html, ?w){|f| f << CODERAY % [File.basename(rb), pre.inner_html]}
  end
end

# Substitute the examples embedded in a given HTML file with the corresponding
# highlighted copies, as created by the rule above. If a short URL exists for a
# permalink, replace the latter with the former.

# Permalink -> Short URL Map
PERMALINKS = YAML.load(File.read 'permalinks.yaml')
PERMALINKS.default_proc= ->(h, permalink) do
  require 'net/http'
  res = Net::HTTP.post_form(URI.parse('http://goo.gl/api/url'), 
                            url: permalink, user: 'toolbar@google.com')
  break permalink unless res.code == '201'
  h[permalink] = res.header['Location']
  open('permalinks.yaml', ?w){|f| YAML.dump(h, f)}
  h[permalink]
end

desc "Replace examples with highlighted versions; shorten permalinks"
task :fixup, [:file]  do |_, a|
  (nok = Nokogiri::HTML(File.read a.file)).css('code.ruby').each do |code|
    f = "#{EX_DIR}/#{code['id'].sub(/^ex\./,'')}.html"
    code.parent.swap(File.read f) if File.exist?(f)
  end
  nok.css('h1 > a').each do |e|
    u = URL + File.basename(a.file.ext('')) + e['href']
    e['href'] = PERMALINKS[u].sub(/^http:/,'')
  end
  open(a.file, ?w) {|f| f << nok.to_s}
end

desc "Validate HTML documents in #{OUT_DIR}"
task :validate_html do
  HTML.map do |f|
    Thread.new do
      mech = Mechanize.new.post(URI.parse('http://validator.nu/?out=json'), 
                                File.read(f), {'Content-Type' => 'text/html'})
      JSON.parse(mech.body.force_encoding('utf-8'))['messages'].
        reject{|m| m['type'] == 'info'}
    end
  end.map(&:value).each_with_index do |msg, i|
    warn "#{HTML[i]} is invalid HTML: #{msg}" unless msg.empty?
  end
end

desc "Validate the XML with RelaxNG"
task :relaxng do
  begin
    sh "xmllint --xinclude --noout --relaxng #{DOCBOOK_RNG} #{BOOK_XML} 2>&1"
  rescue => e
    raise e unless $?.exitstatus == 127
    warn "Skipping RelaxNG validation; xmllint not installed"
  end
end

desc "Validate the XML with NVDL"
task :nvdl do
  begin
    sh "onvdl #{BOOK_XML}"
  rescue => e
    raise e unless $?.exitstatus == 127
    warn "Skipping NVDL validation; oNVDL not installed"
  end
end

task :pdf => [OUT_DIR, BUILD_DIR] do
  XSLTPROC[PDF_XSL]
  html = File.read(PDF_HTML).gsub(/\$git_date/, GIT_DATE)
  open(PDF_HTML, ?w){|f| f << html}
  sh "prince #{PDF_HTML} #{PDF}"
  sh "gzip --best -cn #{PDF} >#{PDF}.gz"
end

desc "Validate the XML with RelaxNG and NVDL"
task :validate_xml => [:relaxng, :nvdl]

desc 'Rebuild then view locally with a web browser'
task :browse => :html do
  sh "./bin/browse #{OUT_DIR}"
end

desc 'Upload current build'
task :rsync  => :html do
  exclude = RSYNC_EXCLUDE.map do |glob|
    "--exclude '#{glob}' "
  end.join
  sh "rsync #{exclude} --delete -vazL out/ ruby:/home/public"
end

desc 'Push to GitHub'
task :push do
  sh 'git push github'
  sh 'git push'
end  

desc 'Validate XML, build HTML, validate HTML, build PDF'
task :default => %w{validate_xml html validate_html pdf}

desc "Rebuild then rsync"
task :upload => %w{clobber default rsync push}
