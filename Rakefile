require 'rake'
require 'nokogiri'
require 'openssl'
require 'net/http'
require 'json'
require 'fileutils'
require 'open-uri'
require 'time'
require 'date'
require 'pp'
require 'zip'
require 'tempfile'
require 'v8'
require 'chronic'
require 'sqlite3'
require 'i18n'
require 'json/minify'

EXTENSION_ID = Nokogiri::XML(File.open('install.rdf')).at('//em:id').inner_text
EXTENSION = EXTENSION_ID.gsub(/@.*/, '')
RELEASE = Nokogiri::XML(File.open('install.rdf')).at('//em:version').inner_text

Dir.mkdir('tmp') unless File.exists?('tmp')

UNICODE_MAPPING = 'tmp/unicode.json'
BIBTEX_GRAMMAR  = Dir["resource/**/*.pegjs"][0]

SOURCES = %w{chrome test/import test/export resource defaults chrome.manifest install.rdf bootstrap.js}
            .collect{|f| File.directory?(f) ?  Dir["#{f}/**/*"] : f}.flatten
            .select{|f| File.file?(f)}
            .reject{|f| f =~ /[~]$/ || f =~ /\.swp$/} + [DATE, UNICODE_MAPPING, BIBTEX_GRAMMAR, DICT]

XPI = "zotero-#{EXTENSION}-#{RELEASE}.xpi"

task :default => XPI do
end

Dir['test/detect/*.*'].sort.each{|test|
  test = File.basename(test).split('.')
  id = "#{test[0].gsub(/[^A-Z]/, '').downcase}/d/#{test[1].to_i}"
  desc "Test: #{test[0]} detect #{test[1]}"
  task id => [DB, XPI] do
    Test.new(test[0], :detect, test[1], test[2])
  end
}

Dir['test/import/*.bib'].sort.each{|test|
  test = File.basename(test).split('.')
  id = "#{test[0].gsub(/[^A-Z]/, '').downcase}/i/#{test[1].to_i}"
  desc "Test: #{test[0]} import #{test[1]}"
  task id => [DB, XPI] do
    Test.new(test[0], :import, test[1])
  end
}
Dir['test/export/*.json'].sort.each{|test|
  test = File.basename(test).split('.')
  id = "#{test[0].gsub(/[^A-Z]/, '').downcase}/e/#{test[1].to_i}"
  desc "Test: #{test[0]} export #{test[1]}"
  task id => [DB, XPI] do
    Test.new(test[0], :export, test[1])
  end
}

task :test => [DB, XPI] do
  # vim -b file and once in vim:
  #:set noeol
  #:wq
  tests = 0
  Dir['test/export/*.json'].sort.each{|test|
    test = File.basename(test).split('.')
    Test.new(test[0], :export, test[1])
    tests += 1
  }

  Dir['test/import/*.bib'].sort.each{|test|
    test = File.basename(test).split('.')
    Test.new(test[0], :import, test[1])
    tests += 1
  }

  Dir['test/detect/*.*'].sort.each{|test|
    test = File.basename(test).split('.')
    Test.new(test[0], :detect, test[1], test[2])
    tests += 1
  }

  puts "\n#{tests} tests ran\n"
end

task :dropbox => :test do
  dropbox = File.expand_path('~/Dropbox')
  Dir["#{dropbox}/*.xpi"].each{|xpi| File.unlink(xpi)}
  FileUtils.cp(XPI, File.join(dropbox, XPI))
end

file XPI => SOURCES do |t|
  Dir['*.xpi'].each{|xpi| File.unlink(xpi)}

  begin
    puts "Creating #{t.name}"
    Zip::File.open(t.name, 'w') do |zipfile|
      t.prerequisites.reject{|f| f=~ /^(test|tmp|resource)\// }.each{|file|
        zipfile.add(file, file)
      }

      zipfile.mkdir('resource/translators')
      TRANSLATORS.each{|translator|
        translator[:source] = "resource/translators/#{translator[:name]}.js"
        zipfile.get_output_stream(translator[:source]){|f|
          f.write((Translator.new(translator)).to_s)
        }
      }
    end
  rescue => e
    File.unlink(t.name) if File.exists?(t.name)
    throw e
  end

#  moz_xpi = "moz-addons-#{t.name}"
#  FileUtils.cp(t.name, moz_xpi)
#
#  Zip::File.open(moz_xpi) {|zf|
#    install_rdf = Nokogiri::XML(zf.file.read('install.rdf'))
#    install_rdf.at('//em:updateURL').unlink
#    zf.file.open('install.rdf', 'w') {|f|
#      f.write install_rdf.to_xml
#    }
#  }
end

file 'update.rdf' => [XPI, 'install.rdf'] do |t|
  update_rdf = Nokogiri::XML(File.open(t.name))
  update_rdf.at('//em:version').content = RELEASE
  update_rdf.at('//RDF:Description')['about'] = "urn:mozilla:extension:#{EXTENSION_ID}"
  update_rdf.xpath('//em:updateLink').each{|link| link.content = "https://raw.github.com/ZotPlus/zotero-#{EXTENSION}/master/#{XPI}" }
  update_rdf.xpath('//em:updateInfoURL').each{|link| link.content = "https://github.com/ZotPlus/zotero-#{EXTENSION}" }
  File.open('update.rdf','wb') {|f| update_rdf.write_xml_to f}
end

task :publish => ['README.md', XPI, 'update.rdf'] do
  sh "git add #{XPI}"
  sh "git commit -am #{RELEASE}"
  sh "git tag #{RELEASE}"
  sh "git push"
end

file 'README.md' => ['install.rdf', 'Rakefile'] do |t|
  puts 'Updating README.md'

  readme = File.open(t.name).read
  readme.gsub!(/\(http[^)]+\.xpi\)/, "(https://github.com/ZotPlus/zotero-#{EXTENSION}/raw/master/#{XPI})")
  readme.gsub!(/\*\*[0-9]+\.[0-9]+\.[0-9]+\*\*/, "**#{RELEASE}**")
  readme.gsub!(/[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}/, DateTime.now.strftime('%Y-%m-%d %H:%M'))
  home = readme if patch =~ /Home\.md$/
  File.open(t.name, 'w'){|f| f.write(readme) }
end

task :release, :bump do |t, args|
  puts `git checkout zotero*.xpi`

  bump = args[:bump] || 'patch'

  release = RELEASE.split('.').collect{|n| Integer(n)}
  release = case bump
            when 'major' then [release[0] + 1, 0, 0]
            when 'minor' then [release[0], release[1] + 1, 0]
            when 'patch' then [release[0], release[1], release[2] + 1]
            else raise "Unexpected release increase #{bump.inspect}"
            end
  release = release.collect{|n| n.to_s}.join('.')

  install_rdf = Nokogiri::XML(File.open('install.rdf'))
  install_rdf.at('//em:version').content = release
  install_rdf.at('//em:updateURL').content = "https://raw.github.com/ZotPlus/zotero-#{EXTENSION}/master/update.rdf"
  File.open('install.rdf','wb') {|f| install_rdf.write_xml_to f}
  puts `git add install.rdf`
  puts "Release set to #{release}. Please publish."
end

#### GENERATED FILES

class Hash
 
  def deep_diff(b)
    a = self
    (a.keys | b.keys).inject({}) do |diff, k|
      if (a[k] || []) != (b[k] || []) && (a[k] || '') != (b[k] || '')
        if a[k].respond_to?(:deep_diff) && b[k].respond_to?(:deep_diff)
          diff[k] = a[k].deep_diff(b[k])
        elsif a[k].is_a?(Array) && b[k].is_a?(Array)
          extra = a[k] - b[k]
          missing = b[k] - a[k]
          if extra.empty? && missing.empty?
            diff[k] = {order: a[k]}
          else
            diff[k] = {missing: missing, extra: extra}
          end
        else
          diff[k] = [a[k], b[k]]
        end
      end
      diff
    end
  end
 
end

class Test
  def pref(key, value)
    tkey = key.sub(/^extensions\.zotero\.translators\./, '')
    return unless tkey != key
    @prefs[tkey] ||= value;
  end

  def script(content, init = false)
    content = content.read unless content.is_a?(String)
    if init
      File.write(@script, content, 0, mode: 'w')
    else
      File.write(@script, content, File.size(@script), mode: 'a')
    end
  end

  def run
    return @ctx.load(@script)
  end

  def initialize(translator, type, id, extension = nil)
    @translator = translator
    @type = type
    @id = id

    puts "\n\nRunning #{type} test #{id} for #{translator}"

    @script = "tmp/#{translator}-#{type}-#{id}.js"
    File.unlink(@script) if File.exists?(@script)

    @prefs = {}
    @options = {}

    options = "test/#{type}/#{translator}.#{id}.options.json"
    @options = JSON.parse(JSON.minify(File.open(options).read)) if File.exist?(options)
    if @options['prefs']
      @options['prefs'].each_pair{|k, v| @prefs[k] = v }
      @options = @options['options'] || {}
    end

    @ctx = V8::Context.new
    @ctx['Zotero'] = self
    @ctx['ZU'] = self
    @ctx['Z'] = self
    @ctx['pref'] = lambda {|this, key, value| pref(key, value)}

    script(File.open(DATE), :init)
    script(File.open('defaults/preferences/defaults.js'))
    script('var __zotero__header__ = ')
    script(File.open("tmp/#{translator}.js"))
    script(File.open('test/utilities.js'))

    @db = SQLite3::Database.new(DB)
    @db.execute('PRAGMA temp_store=MEMORY;')
    @db.execute('PRAGMA journal_mode=MEMORY;')
    @db.execute('PRAGMA synchronous = OFF;')

    case type
      when :import then import
      when :export then export
      when :detect then detect(extension)
      else throw "Unexpected test type #{type}"
    end

    File.unlink(@script) if File.exists?(@script)
  end
  attr_reader :translator, :id, :type
  attr_accessor :Item
  attr_accessor :Date

  def Utilities
    return self
  end

  def getCreatorsForType(type)
    cft = @db.execute('select ct.creatorType
                       from creatorTypes ct
                       join itemTypeCreatorTypes itct on itct.creatorTypeID = ct.creatorTypeID
                       join itemTypes it on it.itemTypeID = itct.itemTypeID
                       where it.typeName = ?', type).collect{|row| row[0]}
    throw type.inspect if cft.empty?
    return cft
  end

  def getHiddenPref(key)
    return @prefs[key];
  end

  def getOption(key)
    return @options[key];
  end

  def testName
    "#{@translator} #{@type} #{@id}"
  end

  def debug(msg)
    puts "\n#{testName}:: #{msg}"
  end

  def read(n)
    return false unless @input && @input != ''

    chunk = @input[0,n]
    @input = @input[n, @input.length]
    return chunk
  end

  def write(str)
    @output += str
  end

  def removeDiacritics(str)
    I18n.transliterate(str)
  end

  def nextCollection
    proc do
      false
    end
  end

  def nextItem
    proc do
      if @items.nil? || @items.empty?
        false
      else
        @items.shift
      end
    end
  end

  attr_accessor :getMonths
  attr_accessor :formatDate
  attr_accessor :cleanAuthor
  attr_accessor :strToDate

  def trim(str)
    throw 'trim: argument must be a string' unless str.is_a?(String)
    return str.gsub(/^\s+/, '').gsub(/\s+$/, '')
  end

  def trimInternal(str)
    @tire ||= Regexp.new('[\xA0\r\n\s]+', nil, 'n')
    throw 'trimInternal: argument must be a string' unless str.is_a?(String)
    return trim(str.gsub(@tire, ' '))
  end

  def text2html(value)
    return value.gsub(/&/, '&amp;').gsub('<', '&lt;').gsub('>', '&gt;')
  end

  def lpad(string, pad, length)
    return "#{string}".rjust(length, pad)
  end

  def complete(item)
    @items << JSON.parse(item)
  end

  private

  def textdiff(a, b)
    _diff = ''
    Tempfile.open('diff') do |da|
      Tempfile.open('diff') do |db|
        da.write(a)
        da.close
        db.write(b)
        db.close
        _diff = `diff -B -u #{da.path.inspect} #{db.path.inspect}`
      end
    end
    _diff.gsub!(/^--- .*?\n/, '')
    _diff.gsub!(/^\+\+\+ .*?\n/, '')
    return _diff
  end

  def export
    input = "test/#{@type}/#{@translator}.#{@id}.json"
    expected = "test/#{@type}/#{@translator}.#{@id}.txt"
    throw "#{input} does not exist" unless File.exists?(input)
    throw "#{expected} does not exist" unless File.exists?(expected)

    @output = ''

    @items = JSON.parse(JSON.minify(File.open(input).read))
    @items.each_with_index{|item, i|
      item['itemID'] = i+1
      item['key'] = "X#{item['itemID']}"
      item['tags'] = item['tags'].collect{|tag| {'tag' => tag}} if item['tags']
      if item['attachments']
        item['attachments'].each{|att|
          att['defaultPath'] = att['path']
          att['localPath'] = att['path']
          att.delete('path')
        }
      end
    }

    expected = File.open(expected).read

    #@ctx.eval(File.open('test/item.js').read)
    script('doExport();')

    run

    File.open("test/#{@type}/#{@translator}.#{@id}.out.txt", 'w'){|f| f.write(@output) }

    diff = textdiff(expected, @output)
    if diff == ''
      puts "#{testName}: passed"
    else
      puts diff
      throw "#{testName}: failed (expected => found)"
    end
  end

  def detect(extension)
    input = "test/#{@type}/#{@translator}.#{@id}.#{extension}"
    @input = File.open(input).read

    script('detectImport();')

    expected = !!(extension.downcase == 'bib')
    found = !!run

    if expected == found
      puts "#{testName}: passed"
    else
      throw "#{testName}: expected #{expected ? '' : 'non-'}bibtex, found #{found ? '' : 'non-'}bibtex"
    end
  end

  def import
    input = "test/#{@type}/#{@translator}.#{@id}.bib"
    expected = "test/#{@type}/#{@translator}.#{@id}.json"
    throw "#{input} does not exist" unless File.exists?(input)
    throw "#{expected} does not exist" unless File.exists?(expected)

    @input = File.open(input).read
    expected = JSON.parse(JSON.minify(File.open(expected).read))
    @items = [];
    script(File.open('test/item.js'))
    script('doImport();')

    run

    File.open("test/#{@type}/#{@translator}.#{@id}.out.json", 'w'){|f| f.write(JSON.pretty_generate(@items)) }

    throw "#{testName}: expected #{expected.size} items, found #{@items.size}" if @items.size != expected.size
    errors = []
    expected.zip(@items).each_with_index{|inout, i|
      expected, found = *inout
      diff = expected.deep_diff(found)
      errors << {item: i + 1, diff: diff} if diff.size != 0
    }
    if errors.size != 0
      pp errors
      throw "#{testName}: failed (expected => found)"
    end

    puts "#{testName}: passed"
  end
end

class Translator
  @@mapping = nil
  @@parser = nil
  @@dict = nil

  def initialize(translator)
    @source = translator[:source]
    @root = File.dirname(@source)
    @_unicode = !!(translator[:unicode])

    @_timestamp = DateTime.now.strftime('%Y-%m-%d %H:%M:%S')
    @_unicode_mapping = Translator.mapping
    @_bibtex_parser = Translator.parser
    @_dict = Translator.dict
    @_release = RELEASE
    get_testcases
  end
  attr_reader :_id, :_label, :_timestamp, :_release, :_unicode, :_unicode_mapping, :_bibtex_parser, :_dict, :_testcases

  def self.dict
    @@dict ||= File.open(DICT).read
  end

  def self.parser
    if @@parser.nil?
      p = 'tmp/parser.js'
      puts "Generating #{p.inspect} from #{BIBTEX_GRAMMAR.inspect}"
      puts `pegjs -e BibTeX #{BIBTEX_GRAMMAR.inspect} #{p.inspect}`
      exit unless $? == 0
      @@parser = File.open(p).read
    end
    return @@parser
  end

  def self.mapping
    if @@mapping.nil?
      mapping = JSON.parse(open(UNICODE_MAPPING).read)

      u2l = {
        unicode: {
          math: [],
          map: {}
        },
        ascii: {
          math: [],
          map: {}
        }
      }

      l2u = { }

      mapping.each_pair{|key, repl|
        # need to figure something out for this. This has the form X<combining char>, which needs to be transformed to 
        # \combinecommand{X}
        #raise value if value =~ /LECO/

        latex = [repl['latex']]
        case repl['latex']
          when /^(\\[a-z][^\s]*)\s$/i, /^(\\[^a-z])\s$/i  # '\ss ', '\& ' => '{\\s}', '{\&}'
            latex << "{#{$1}}"
          when /^(\\[^a-z]){(.)}$/                       # '\"{a}' => '\"a'
            latex << "#{$1}#{$2}"
          when /^(\\[^a-z])(.)\s*$/                       # '\"a " => '\"{a}'
            latex << "#{$1}{#{$2}}"
          when /^{(\\[.]+)}$/                             # '{....}' '.... '
            latex << "#{$1} "
        end

        # prefered option is braces-over-traling-space because of miktex bug that doesn't ignore spaces after commands
        latex.sort!{|a, b|
          nsa = !(a =~ /\s$/)
          nsb = !(a =~ /\s$/)
          ba = a.gsub(/[^{]/, '')
          bb = b.gsub(/[^{]/, '')
          if nsa == nsb
            bb <=> ba
          elsif nsa
            -1
          elsif nsb
            1
          else
            a <=> b
          end
        }

        if key =~ /^[\x20-\x7E]$/ # an ascii character that needs translation? Probably a TeX special character
          u2l[:unicode][:map][key] = latex[0]
          u2l[:unicode][:math] << key if repl['math']
        end

        u2l[:ascii][:map][key] = latex[0]
        u2l[:ascii][:math] << key if repl['math']

        latex.each{|ltx|
          l2u[ltx] = key if ltx =~ /\\/
        }
      }

      [:ascii, :unicode].each{|map|
        u2l[map][:math] = '/(' + u2l[map][:math].collect{|key| key.gsub(/([\\\^\$\.\|\?\*\+\(\)\[\]\{\}])/, '\\\\\1') }.join('|') + ')/g'
        u2l[map][:text] = '/' + u2l[map][:map].keys.collect{|key| key.gsub(/([\\\^\$\.\|\?\*\+\(\)\[\]\{\}])/, '\\\\\1') }.join('|') + '/g'
      }

      @@mapping = "
        var LaTeX = {
          regex: {
            unicode: {
              math: #{u2l[:unicode][:math]},
              text: #{u2l[:unicode][:text]}
            },

            ascii: {
              math: #{u2l[:ascii][:math]},
              text: #{u2l[:ascii][:text]}
            }
          },

          toLaTeX: #{JSON.pretty_generate(u2l[:ascii][:map])},
          toUnicode: #{JSON.pretty_generate(l2u)}
        };
        "
    end

    return @@mapping
  end

  def get_testcases
    @_testcases = []
    Dir["test/import/#{File.basename(@source, File.extname(@source)) + '.*.bib'}"].sort.each{|test|
      @_testcases << {type: 'import', input: File.open(test).read, items: JSON.parse(JSON.minify(File.open(test.gsub(/\.bib$/, '.json')).read))}
    }
    @_testcases = JSON.pretty_generate(@_testcases)
  end

  def _include(partial)
    return render(File.read(File.join(@root, File.basename(partial))))
  end

  def to_s
    puts "Creating #{@source}"

    #code = File.open(@template, 'rb', :encoding => 'utf-8').read
    js = File.open(@source, 'r').read

    header = nil
    start = js.index('{')
    length = 2
    while start && length < 1024
      begin
        header = JSON.parse(js[start, length])
        break
      rescue JSON::ParserError
        header = nil
        length += 1
      end
    end

    raise "No header in #{@template}" unless header

    @_id = header['translatorID']
    @_label = header['label']
    js = render(js)

    File.open(File.join('tmp', File.basename(@source)), 'w'){|f| f.write(js) }

    return js
  end

  def render(template)
    return template.gsub(/\/\*= (.*?) =\*\//){|match, command|
      arguments = $1.split
      command = arguments.shift
      self.send("_#{command}".intern, *arguments)
    }
  end
end

file 'tmp/zotero.sql' do
  download('https://raw.githubusercontent.com/zotero/zotero/4.0/resource/schema/system.sql', 'tmp/zotero.sql')
end

file DB => 'tmp/zotero.sql' do |t|
  File.unlink(DB) if File.exists?(DB)

  db = SQLite3::Database.new(DB)
  db.execute('PRAGMA temp_store=MEMORY;')
  db.execute('PRAGMA journal_mode=MEMORY;')
  db.execute('PRAGMA synchronous = OFF;')
  db.execute_batch(File.open(t.prerequisites[0]).read)
  db.close
end

file DATE do
  download('https://raw.githubusercontent.com/zotero/zotero/4.0/chrome/content/zotero/xpcom/date.js', DATE)
end

file UNICODE_MAPPING => 'Rakefile' do |t|
  begin
    xml = File.join(File.dirname(t.name), File.basename(t.name, File.extname(t.name)) + '.xml')
    download('http://www.w3.org/2003/entities/2007xml/unicode.xml', xml)

    mapping = Nokogiri::XML(open(xml))
    puts mapping.errors

    mapping.at('//charlist') << "
      <character id='U0026' dec='38' mode='text' type='punctuation'><latex>\\&</latex></character>
      <character id='UFFFD' dec='239-191-189' mode='text' type='punctuation'><latex>\\dbend</latex></character>
    "

    {
      "\\textdollar"        => "\\$",
      "\\textquotedblleft"  => "``",
      "\\textquotedblright" => "''",
      "\\textasciigrave"    => "`",
      "\\textquotesingle"   => "'",
      "\\space"             => ' '
    }.each_pair{|ist, soll|
      nodes = mapping.xpath(".//latex[normalize-space(text())='#{ist}']")
      next unless nodes
      nodes.each{|node| node.content = soll }
    }

    json = {}
    mapping.xpath('//character[@dec and latex]').each{|char|
      id = char['dec'].to_s.split('-').collect{|i| Integer(i)}
      key = id.pack('U' * id.size)
      value = char.at('.//latex').inner_text
      mathmode = (char['mode'] == 'math')

      case key
        when '_', '}', '{', '[', ']'
          value = "\\" + key
          mathmode = false
        when "\u00A0"
          value = ' '
          mathmode = false
      end

      next if key =~ /^[\x20-\x7E]$/ && ! %w{# $ % & ~ _ ^ { } [ ] > < \\}.include?(key)
      next if key == value && !mathmode

      # need to figure something out for this. This has the form X<combining char>, which needs to be transformed to 
      # \combinecommand{X}
      #raise value if value =~ /LECO/

      json[key] = {latex: value, math: mathmode}
    }

    #File.open(t.name,'w') {|f| mapping.write_xml_to f}
    File.open(t.name,'w') {|f| f.write(json.to_json) }
  rescue => e
    File.rename(t.name, t.name + '.err') if File.exists?(t.name)
    throw e
  end
end

### UTILS

def download(url, file)
  puts "Downloading #{url} to #{file}"
  options = {}
  options['If-Modified-Since'] = File.mtime(file).rfc2822 if File.exists?(file)
  begin
    open(url, options) {|remote|
      open(file, 'wb'){|local| local.write(remote.read) }
    }
  rescue OpenURI::HTTPError => e
  end
end
