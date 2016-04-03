# GimmeGet
import java.net.*;
import java.io.*;
import java.util.*;
import java.text.*;

import org.w3c.dom.*;
import org.xml.sax.InputSource;
import javax.xml.parsers.*;
import javax.xml.xpath.*;
import javax.xml.transform.*;
import javax.xml.transform.dom.*;
import javax.xml.transform.stream.*;

public class classifySample {
	private static String classifyURL = "http://classify.oclc.org/classify2/Classify?";
	
	private void go(String args[]) {
		String paramType=null, paramValue=null, summaryInd="false";
		
		if (args.length >= 2) {
			paramType = args[0];
			paramValue = args[1];
			if (args.length > 2)
				summaryInd = args[2];
		} else {
			System.out.println("improper usage");
			usage();
			System.exit(-1);
		}
		System.out.println("Searching for: " + paramType + "=" + paramValue);
		
		Document doc = null;
		String nextURL = null;
		try {
			if (summaryInd.equals("true")) 
				nextURL = classifyURL + paramType + "=" + java.net.URLEncoder.encode(paramValue, "UTF-8") + "&summary=true";
			else
				nextURL = classifyURL + paramType + "=" + java.net.URLEncoder.encode(paramValue, "UTF-8");
			URL url = new URL(nextURL);			
			InputStream in = url.openStream();
			doc = parseXML(in, false, false);
			in.close();
		} catch (Exception e) {
			System.err.println(e);
			e.printStackTrace();
		}
				
		XPath xpath = XPathFactory.newInstance().newXPath();
		Node respNode = getNode(xpath, doc, "//response");
		if (respNode != null) {
			String respCode = getAttribute(xpath, respNode, "@code");
			System.out.println("response code=" + respCode);
			if (respCode.equals("0") || respCode.equals("2")) printRecommendations(doc, xpath);
			else if (respCode.equals("4")) printEditions(doc, xpath);
		}
	}
	
	private void printRecommendations(Document doc, XPath xpath) {
		try {
			System.out.println("DDC");
			String expression = "//recommendations/ddc/mostPopular";
			NodeList list = (NodeList) xpath.evaluate(expression, doc, XPathConstants.NODESET);
			if (list.getLength() > 0)
				System.out.println("   Most Popular");
			for (int i=0; i<list.getLength(); i++) {
				Node node = list.item(i);
				String sfa = getAttribute(xpath, node, "@sfa");
				String nsfa = getAttribute(xpath, node, "@nsfa");
				String holdings = getAttribute(xpath, node, "@holdings");
				System.out.println("      class=" + sfa + " normalized class=" + nsfa + " holdings=" + holdings);
			}
			
			System.out.println("LCC");
			expression = "//recommendations/lcc/mostPopular";
			list = (NodeList) xpath.evaluate(expression, doc, XPathConstants.NODESET);
			if (list.getLength() > 0)
				System.out.println("   Most Popular");
			for (int i=0; i<list.getLength(); i++) {
				Node node = list.item(i);
				String sfa = getAttribute(xpath, node, "@sfa");
				String nsfa = getAttribute(xpath, node, "@nsfa");
				String holdings = getAttribute(xpath, node, "@holdings");
				System.out.println("      class=" + sfa + " normalized class=" + nsfa + " holdings=" + holdings);
			}
		} catch (Exception e) {
			System.err.println(e);
			e.printStackTrace();
		}
	}
	
	private void printEditions(Document doc, XPath xpath) {
		try {
			String expression = "//editions/edition";
			NodeList list = (NodeList) xpath.evaluate(expression, doc, XPathConstants.NODESET);
			for (int i=0; i<list.getLength(); i++) {
				Node node = list.item(i);
				String title = getAttribute(xpath, node, "@title");
				String author = getAttribute(xpath, node, "@author");
				String date = getAttribute(xpath, node, "@date");
				String editions = getAttribute(xpath, node, "@editions");
				String format = getAttribute(xpath, node, "@format");
				String owi = getAttribute(xpath, node, "@owi");
				System.out.println("title=" + title + " author=" + author + " editions=" + editions + " date=" + date + " format=" + format + " owi=" + owi);
			}
		} catch (Exception e) {
			System.err.println(e);
			e.printStackTrace();
		}
	}
		
	private Document parseXML(InputStream in, boolean validating, boolean namespaceAware) {
		Document doc = null;
		try {
			DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
			factory.setNamespaceAware(namespaceAware);
			factory.setValidating(validating);
			DocumentBuilder builder = factory.newDocumentBuilder();	
			doc = builder.parse(in);
		    } catch (Exception e) {
			System.err.println(e);
			e.printStackTrace();
		    }
		    return doc;
	}
	
	private String getStringFromDocument(Document doc) {
		try {
			DOMSource domSource = new DOMSource(doc);
			StringWriter writer = new StringWriter();
			StreamResult result = new StreamResult(writer);
			TransformerFactory tf = TransformerFactory.newInstance();
			Transformer transformer = tf.newTransformer();
			transformer.setOutputProperty("{http://xml.apache.org/xalan}indent-amount", "2");
			transformer.setOutputProperty("{http://xml.apache.org/xslt}indent-amount", "2");
			transformer.setOutputProperty(OutputKeys.INDENT, "yes");
			transformer.transform(domSource, result);
			return writer.toString();
		} catch(TransformerException ex) {
			System.err.println(ex);
			ex.printStackTrace();
			return null;
		}
	}
	
	private Node getNode(XPath xpath, Node parent, String expression) {
		Node node = null;
		try {
			node = (Node) xpath.evaluate(expression, parent, XPathConstants.NODE);
		} catch (Exception e) {
			System.err.println(e);
			e.printStackTrace();
		}
		return node;	
	}
		
	private String getNodeValue(XPath xpath, Node parent, String expression) {
		String nodeValue = null;
		try {
			Node node = (Node) xpath.evaluate(expression, parent, XPathConstants.NODE);
			if (node != null)
				nodeValue = node.getTextContent();
		} catch (Exception e) {
			System.err.println(e);
			e.printStackTrace();
		}
		return nodeValue;	
	}
	
	private String getAttribute(XPath xpath, Node node, String expression) {
		String attrValue = null;
		try {
			NamedNodeMap attrs = node.getAttributes();
			for (int i = 0; i < attrs.getLength(); i++) {
				Node a = attrs.item(i);
				if (a.getNodeName().equals(expression.substring(1)))
					attrValue = a.getNodeValue();
			}
		} catch (Exception e) {
			System.err.println(e);
			e.printStackTrace();
		}
		
		return attrValue;
	}
	
	private static final void usage() {
		System.out.println("usage: java classifySample <param-type> <param-value> [summary ind]");
		System.out.println("example: java classifySample oclc 166382351 true");
	}
	
	public static void main(String[] args) {
		new classifySample().go(args);
	}
}
